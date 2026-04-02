---
title: Create and configure a Microsoft Discovery workspace and resources
description: Learn how to plan your network, create a workspace with network isolation, and deploy bookshelves, supercomputers, storage, projects, and AI model deployments in Microsoft Discovery.
ms.service: azure
ms.topic: how-to
ms.date: 03/30/2026
ms.author: umamm
author: umamm
---

# Create and configure a Microsoft Discovery workspace and resources

A Microsoft Discovery workspace is the top-level resource for AI-powered scientific research. It provides a unified environment for AI agent execution, tool orchestration, knowledge management, and high-performance computing. When you create a workspace, Azure provisions a managed resource group containing the supporting infrastructure your workloads need, including database, storage, AI services, and key vault resources.

This guide walks you through the end-to-end process of creating a workspace and its associated resources, including bookshelves, supercomputers, storage, projects, and AI model deployments.

## Overview

A fully configured Microsoft Discovery environment consists of several resource types that work together:

| Resource | Purpose |
|----------|---------|
| **Workspace** | Top-level container that provisions a managed resource group with supporting services (Cosmos DB, Storage, AI Foundry, Key Vault). |
| **Project** | A subresource of a workspace that creates an AI Foundry project for organizing experiments and deployments. |
| **Chat model deployment** | A subresource of a workspace that deploys AI models for conversational and research tasks. |
| **Bookshelf** | A knowledge management resource that enables document indexing and retrieval. Provisions its own managed resource group with SQL, Storage, AI Foundry, and search services. |
| **Supercomputer** | An AKS-based high-performance computing resource for large-scale scientific workloads. Provisions a managed resource group with AKS and Key Vault. |
| **Storage** | A NetApp Files-based storage resource for high-performance file access. |
| **Storage container** | A reference to an Azure Blob Storage account for data ingestion and output. |

## Prerequisites

Before you begin, ensure you have:

- An Azure subscription. If you don't have one, [create a free account](https://azure.microsoft.com/free/).
- The **Microsoft.Discovery** resource provider registered on your subscription:

  ```azurecli
  az provider register --namespace Microsoft.Discovery
  ```

- Azure CLI version 2.50 or later. Run `az --version` to check. To install or upgrade, see [Install the Azure CLI](/cli/azure/install-azure-cli).
- A resource group in a [supported region](#supported-regions) for your workspace resources.
- A user-assigned managed identity (UAMI) for workload authentication. The UAMI is used by the workspace to authenticate against managed resources on your behalf.

  ```azurecli
  az identity create \
    --name "{uamiName}" \
    --resource-group "{resourceGroupName}" \
    --location "{region}"
  ```

- **For network-isolated deployments** (recommended for production):
  - A virtual network with dedicated subnets (see [Plan your virtual network](#plan-your-virtual-network)).
  - The **Discovery NSP Perimeter Joiner** custom role assigned to the Discovery control-plane service principal (`92c174ac-8e41-4815-a1b7-d81b19ab03ce`) at the subscription scope. See [Configure network security](how-to-configure-network-security.md#step-1-assign-the-nsp-perimeter-joiner-role) for the role definition and assignment instructions.
  - The **Microsoft Discovery Platform Administrator (Preview)** or **Microsoft Discovery Platform Contributor (Preview)** role assigned to your user or service principal on the target resource group.

> [!IMPORTANT]
> Network isolation is recommended for production workloads. Without network isolation, managed resources are accessible over public endpoints.

## Supported regions

Microsoft Discovery is available in the following Azure regions:

- East US
- East US 2
- UK South
- Sweden Central

> [!NOTE]
> Region availability may change. Check the [Azure products by region](https://azure.microsoft.com/explore/global-infrastructure/products-by-region/) page for the latest information. Network isolation is supported in all Discovery regions.

## Plan your virtual network

When you deploy Discovery resources with network isolation, each resource type requires one or more dedicated subnets. The following table summarizes the recommended subnet layout for a complete Discovery deployment.

| Subnet purpose | Used by | Recommended CIDR | Minimum CIDR | Delegation required | Notes |
|----------------|---------|-------------------|--------------|---------------------|-------|
| **Agent subnet** | Workspace | `/24` | `/26` | None | Hosts AI agent workloads. Size based on expected concurrent agent count. |
| **Private endpoint subnet** | Workspace, Bookshelf | `/24` | `/26` | None | Hosts private endpoints for managed resources. Shared across workspace and bookshelf. |
| **Workspace subnet** | Workspace | `/24` | `/26` | None | Hosts workspace platform services. |
| **Bookshelf search subnet** | Bookshelf | `/24` | `/27` | None | Hosts search and indexing services for knowledge management. |
| **Supercomputer subnet** | Supercomputer | `/24` | `/24` | None | Hosts the supercomputer cluster. |
| **Nodepool subnet** | Supercomputer nodepools | `/21` | `/23` | None | Hosts nodepool compute nodes. Size generously for node scaling. A `/21` supports approximately 500 nodes. |
| **Storage NetApp subnet** | Storage | `/24` | `/28` | `Microsoft.NetApp/volumes` | Delegated subnet for Azure NetApp Files volumes. |

> [!TIP]
> For a proof-of-concept deployment, a single `/22` virtual network can accommodate all subnets. For production workloads, plan a `/16` or larger address space to allow for growth.

### Example virtual network setup

Create a virtual network with the recommended subnets:

```azurecli
# Create the virtual network
az network vnet create \
  --name "discovery-vnet" \
  --resource-group "{resourceGroupName}" \
  --location "{region}" \
  --address-prefixes "10.0.0.0/16"

# Create subnets
az network vnet subnet create \
  --name "agent-subnet" \
  --vnet-name "discovery-vnet" \
  --resource-group "{resourceGroupName}" \
  --address-prefixes "10.0.1.0/24"

az network vnet subnet create \
  --name "private-endpoint-subnet" \
  --vnet-name "discovery-vnet" \
  --resource-group "{resourceGroupName}" \
  --address-prefixes "10.0.2.0/24"

az network vnet subnet create \
  --name "workspace-subnet" \
  --vnet-name "discovery-vnet" \
  --resource-group "{resourceGroupName}" \
  --address-prefixes "10.0.3.0/24"

az network vnet subnet create \
  --name "bookshelf-search-subnet" \
  --vnet-name "discovery-vnet" \
  --resource-group "{resourceGroupName}" \
  --address-prefixes "10.0.4.0/24"

az network vnet subnet create \
  --name "supercomputer-subnet" \
  --vnet-name "discovery-vnet" \
  --resource-group "{resourceGroupName}" \
  --address-prefixes "10.0.8.0/24"

az network vnet subnet create \
  --name "nodepool-subnet" \
  --vnet-name "discovery-vnet" \
  --resource-group "{resourceGroupName}" \
  --address-prefixes "10.0.9.0/21"

az network vnet subnet create \
  --name "storage-netapp-subnet" \
  --vnet-name "discovery-vnet" \
  --resource-group "{resourceGroupName}" \
  --address-prefixes "10.0.16.0/24" \
  --delegations "Microsoft.NetApp/volumes"
```

## Create a workspace

A workspace is the foundation of your Discovery environment. You can create a workspace with or without network isolation.

### Create a workspace with network isolation (recommended)

# [Azure CLI](#tab/azure-cli)

```azurecli
az rest --method PUT \
  --uri "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Discovery/workspaces/{workspaceName}?api-version=2026-02-01-preview" \
  --body '{
    "location": "{region}",
    "tags": {
      "networkIsolation": "true",
      "SkipAssociateKeyVaultToNsp": "true"
    },
    "properties": {
      "agentSubnetId": "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Network/virtualNetworks/{vnetName}/subnets/agent-subnet",
      "privateEndpointSubnetId": "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Network/virtualNetworks/{vnetName}/subnets/private-endpoint-subnet",
      "workspaceSubnetId": "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Network/virtualNetworks/{vnetName}/subnets/workspace-subnet",
      "workspaceIdentity": {
        "id": "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.ManagedIdentity/userAssignedIdentities/{uamiName}"
      }
    }
  }'
```

# [Azure portal](#tab/portal)

1. Sign in to the [Azure portal](https://portal.azure.com).
2. In the search bar, type **Microsoft Discovery** and select it from the results.
3. Select **+ Create** > **Workspace**.
4. On the **Basics** tab:
   - Select your **Subscription** and **Resource group**.
   - Enter a **Workspace name** (must be globally unique, 3–24 characters, lowercase alphanumeric and hyphens only).
   - Select a **Region**.
5. On the **Networking** tab:
   - Select **Enable network isolation**.
   - Select the **Virtual network** you created.
   - Map each subnet role (agent, private endpoint, workspace) to the corresponding subnet.
6. On the **Identity** tab:
   - Select the **User-assigned managed identity** to use for workload authentication.
7. On the **Tags** tab, add any custom tags for your organization.
8. Select **Review + create**, review the configuration summary, then select **Create**.

---

> [!NOTE]
> Workspace provisioning typically takes 10–20 minutes. The operation creates a managed resource group and deploys supporting infrastructure on your behalf.

### Create a workspace without network isolation

For development and testing scenarios, you can create a workspace without network isolation. Omit the `tags` for network isolation and the subnet properties:

# [Azure CLI](#tab/azure-cli)

```azurecli
az rest --method PUT \
  --uri "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Discovery/workspaces/{workspaceName}?api-version=2026-02-01-preview" \
  --body '{
    "location": "{region}",
    "properties": {
      "workspaceIdentity": {
        "id": "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.ManagedIdentity/userAssignedIdentities/{uamiName}"
      }
    }
  }'
```

# [Azure portal](#tab/portal)

Follow the same steps as the network-isolated deployment, but on the **Networking** tab, leave **Enable network isolation** unchecked.

---

> [!IMPORTANT]
> Workspaces created without network isolation expose managed resources over public endpoints. Use this configuration only for non-production workloads.

## Create a bookshelf

A bookshelf provides knowledge management capabilities, including document indexing and semantic search. For detailed setup instructions, see [Create a bookshelf](how-to-create-bookshelf.md).

> [!TIP]
> If you're deploying a bookshelf alongside a workspace with network isolation, use the same virtual network and share the private endpoint subnet. The bookshelf requires its own dedicated search subnet.

## Create a supercomputer

A supercomputer provides high-performance computing for large-scale scientific workloads. The compute cluster is **VNet-injected** - it runs directly in your virtual network subnet.

> [!IMPORTANT]
> The supercomputer subnet requires a `/24` CIDR (256 IPs) and must be in the **same VNet** as the workspace subnets.

> [!NOTE]
> **Known limitation:** The supercomputer's AKS API server has a public FQDN. Workload traffic stays within the virtual network, but the Kubernetes API server endpoint is publicly accessible. Private cluster support is planned for a future release.

### Create the supercomputer

# [Azure CLI](#tab/azure-cli)

```azurecli
az rest --method PUT \
  --uri "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Discovery/supercomputers/{supercomputerName}?api-version=2026-02-01-preview" \
  --body '{
    "location": "{region}",
    "properties": {
      "subnetId": "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Network/virtualNetworks/{vnetName}/subnets/sc-aks",
      "identities": {
        "clusterIdentity": { "id": "{clusterIdentityResourceId}" },
        "kubeletIdentity": { "id": "{kubeletIdentityResourceId}" },
        "workloadIdentities": { "{uamiResourceId}": {} }
      }
    }
  }'
```

# [Azure portal](#tab/portal)

1. In the Azure portal, search for **Microsoft Discovery** and select **+ Create** > **Supercomputer**.
2. Select your subscription, resource group, name, and region.
3. Select the VNet-injected subnet for the compute cluster.
4. Select the cluster identity, kubelet identity, and workload identities.
5. Select **Review + create**, then **Create**.

---

### Add a nodepool

After the supercomputer is created, add a nodepool to provide compute capacity:

```azurecli
az rest --method PUT \
  --uri "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Discovery/supercomputers/{supercomputerName}/nodepools/{nodepoolName}?api-version=2026-02-01-preview" \
  --body '{
    "location": "{region}",
    "properties": {
      "vmSize": "Standard_D16s_v5",
      "count": 3,
      "mode": "User"
    }
  }'
```

Nodepools inherit the virtual network injection from the supercomputer - all nodes run in the same subnet.

## Create storage (optional)

Discovery storage resources use Azure NetApp Files to provide high-performance NFS file access. This resource is **optional** - workspaces and bookshelves function without it.

> [!TIP]
> Consider creating a NetApp storage resource only if your workloads require high-throughput NFS file access for large datasets (for example, molecular simulation input/output files, training data corpora, or shared research datasets). NetApp Files incurs ongoing costs regardless of usage. For simpler blob storage needs, use a [storage container](#create-a-storage-container) instead.

> [!IMPORTANT]
> Before creating a Discovery storage resource, ensure that your subscription has the **Microsoft.NetApp** resource provider registered and that the storage subnet is delegated to `Microsoft.NetApp/volumes`.

# [Azure CLI](#tab/azure-cli)

```azurecli
az rest --method PUT \
  --uri "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Discovery/storages/{storageName}?api-version=2025-07-01-preview" \
  --body '{
    "location": "{region}",
    "properties": {
      "store": {
        "kind": "AzureNetApp",
        "subnetId": "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Network/virtualNetworks/{vnetName}/subnets/storage-netapp-subnet"
      }
    }
  }'
```

# [Azure portal](#tab/portal)

1. In the Azure portal, search for **Microsoft Discovery** and select it.
2. Select **+ Create** > **Storage**.
3. On the **Basics** tab, select your subscription, resource group, enter a name, and select a region.
4. On the **Storage configuration** tab:
   - Select **AzureNetApp** as the storage kind.
   - Select the delegated NetApp subnet.
5. Select **Review + create**, then select **Create**.

---

## Link a supercomputer to a workspace

To use a supercomputer for workloads within a workspace, you link it by updating the workspace with the supercomputer's resource ID. This is done by performing a re-PUT (update) of the workspace resource.

# [Azure CLI](#tab/azure-cli)

```azurecli
az rest --method PUT \
  --uri "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Discovery/workspaces/{workspaceName}?api-version=2026-02-01-preview" \
  --body '{
    "location": "{region}",
    "tags": {
      "networkIsolation": "true",
      "SkipAssociateKeyVaultToNsp": "true"
    },
    "properties": {
      "agentSubnetId": "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Network/virtualNetworks/{vnetName}/subnets/agent-subnet",
      "privateEndpointSubnetId": "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Network/virtualNetworks/{vnetName}/subnets/private-endpoint-subnet",
      "workspaceSubnetId": "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Network/virtualNetworks/{vnetName}/subnets/workspace-subnet",
      "workspaceIdentity": {
        "id": "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.ManagedIdentity/userAssignedIdentities/{uamiName}"
      },
      "supercomputerIds": [
        "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Discovery/supercomputers/{supercomputerName}"
      ]
    }
  }'
```

# [Azure portal](#tab/portal)

1. Navigate to your workspace in the Azure portal.
2. Under **Settings**, select **Linked resources**.
3. Select **+ Link supercomputer**.
4. Choose the supercomputer from the list and select **Save**.

---

> [!TIP]
> You can link multiple supercomputers to a single workspace by including multiple resource IDs in the `supercomputerIds` array. The re-PUT must include all existing workspace properties to avoid overwriting the current configuration.

## Create a project

A project is a sub-resource of a workspace that creates an AI Foundry project for organizing experiments, model deployments, and data assets.

# [Azure CLI](#tab/azure-cli)

```azurecli
az rest --method PUT \
  --uri "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Discovery/workspaces/{workspaceName}/projects/{projectName}?api-version=2026-02-01-preview" \
  --body '{
    "properties": {}
  }'
```

# [Azure portal](#tab/portal)

1. Navigate to your workspace in the Azure portal.
2. Under **Resources**, select **Projects**.
3. Select **+ Create project**.
4. Enter a project name and select **Create**.

---

## Create a chat model deployment

A chat model deployment provisions an AI model under your workspace for conversational and research tasks.

# [Azure CLI](#tab/azure-cli)

```azurecli
az rest --method PUT \
  --uri "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Discovery/workspaces/{workspaceName}/chatModelDeployments/{deploymentName}?api-version=2026-02-01-preview" \
  --body '{
    "properties": {
      "model": {
        "name": "{modelName}",
        "version": "{modelVersion}"
      }
    }
  }'
```

# [Azure portal](#tab/portal)

1. Navigate to your workspace in the Azure portal.
2. Under **Resources**, select **Chat model deployments**.
3. Select **+ Create deployment**.
4. Choose the model, enter a deployment name and version, then select **Create**.

---

> [!NOTE]
> Available models and versions depend on your region and subscription quota. Check the workspace resource page for supported model options.

## Create a storage container

A storage container creates a reference to an Azure Blob Storage account, enabling data ingestion and output for workspace workloads.

### Prerequisites for storage containers

Ensure you have an existing Azure Storage account with blob storage enabled.

# [Azure CLI](#tab/azure-cli)

```azurecli
az rest --method PUT \
  --uri "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Discovery/workspaces/{workspaceName}/storageContainers/{containerName}?api-version=2026-02-01-preview" \
  --body '{
    "properties": {
      "storageStore": {
        "kind": "AzureStorageBlob",
        "storageAccountId": "/subscriptions/{subscriptionId}/resourceGroups/{storageResourceGroupName}/providers/Microsoft.Storage/storageAccounts/{storageAccountName}"
      }
    }
  }'
```

# [Azure portal](#tab/portal)

1. Navigate to your workspace in the Azure portal.
2. Under **Resources**, select **Storage containers**.
3. Select **+ Create storage container**.
4. Select **Azure Storage Blob** as the kind.
5. Choose the target storage account and select **Create**.

---

## Verify workspace provisioning

After creating your workspace and associated resources, verify that everything provisioned successfully.

### Check workspace status

# [Azure CLI](#tab/azure-cli)

```azurecli
az rest --method GET \
  --uri "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Discovery/workspaces/{workspaceName}?api-version=2026-02-01-preview" \
  --query "{name:name, provisioningState:properties.provisioningState, dataPlaneUri:properties.dataPlaneUri}"
```

# [Azure portal](#tab/portal)

1. Navigate to your workspace in the Azure portal.
2. On the **Overview** page, check the **Status** field. A successfully provisioned workspace shows **Succeeded**.

---

The workspace is ready when `provisioningState` shows `Succeeded`.

### Check bookshelf status

```azurecli
az rest --method GET \
  --uri "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Discovery/bookshelves/{bookshelfName}?api-version=2026-02-01-preview" \
  --query "{name:name, provisioningState:properties.provisioningState}"
```

### Check supercomputer status

```azurecli
az rest --method GET \
  --uri "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Discovery/supercomputers/{supercomputerName}?api-version=2026-02-01-preview" \
  --query "{name:name, provisioningState:properties.provisioningState}"
```

### Access the workspace

After provisioning completes, you can access your workspace through:

- **Data-plane API**: `https://{workspaceName}.workspace.discovery.azure.com`
- **Discovery Studio UI**: `https://studio.discovery.microsoft.com/workspaces/{workspaceName}`

> [!TIP]
> Bookmark the Discovery Studio URL for quick access to the workspace UI, where you can manage projects, run experiments, and monitor workloads.

## Next steps

- [Configure network security](how-to-configure-network-security.md) - Set up network hardening and private endpoints for your workspace.
- [Manage Supercomputer and Nodepools](how-to-manage-supercomputers.md) - Submit jobs to your linked supercomputer.
