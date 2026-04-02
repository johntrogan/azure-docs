---
title: Create a Microsoft Discovery bookshelf
description: Learn how to create and configure a Microsoft Discovery bookshelf for knowledge management with data ingestion, indexing, and retrieval capabilities.
ms.service: azure
ms.topic: how-to
ms.date: 03/30/2026
ms.author: umamm
author: umamm
ms.custom: networking, private-link
---

# Create a Microsoft Discovery bookshelf

A Microsoft Discovery bookshelf provides knowledge management capabilities for your scientific research workloads. It enables data ingestion, indexing, and retrieval so your workspace agents can access and reason over structured and unstructured data.

## What a bookshelf provides

When you create a bookshelf, the Discovery control plane provisions managed resources on your behalf, including:

- **Search and indexing** - for fast data retrieval across your knowledge base
- **AI services** - for intelligent document processing and embeddings
- **Secure storage** - for ingested data and index artifacts
- **Compute environment** - for data processing pipelines

> [!NOTE]
> All managed resources are provisioned in a managed resource group within your subscription. You retain visibility into these resources but the Discovery service manages their lifecycle.

## Prerequisites

- An Azure subscription with the **Microsoft.Discovery** resource provider registered.
- Azure CLI 2.50+ or Azure PowerShell 10.0+.
- A user-assigned managed identity (UAMI) **in the same region** as the bookshelf.
- For network-isolated bookshelves:
  - A virtual network with dedicated subnets for search services and private endpoints.
  - The **Discovery NSP Perimeter Joiner** custom role assigned. See [Configure network security](how-to-configure-network-security.md#step-1-assign-the-nsp-perimeter-joiner-role).

> [!IMPORTANT]
> The UAMI must be in the **same Azure region** as the bookshelf. A location mismatch causes the creation to fail.

## Create a bookshelf with network isolation

Network-isolated bookshelves protect all managed resources with Network Security Perimeters and private endpoints.

# [Azure CLI](#tab/azure-cli)

```azurecli
az rest --method PUT \
  --uri "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Discovery/bookshelves/{bookshelfName}?api-version=2026-02-01-preview" \
  --body '{
    "location": "{region}",
    "tags": {
      "networkIsolation": "true",
      "SkipAssociateKeyVaultToNsp": "true"
    },
    "properties": {
      "searchSubnetId": "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Network/virtualNetworks/{vnetName}/subnets/{searchSubnet}",
      "privateEndpointSubnetId": "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Network/virtualNetworks/{vnetName}/subnets/{peSubnet}",
      "workloadIdentities": {
        "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.ManagedIdentity/userAssignedIdentities/{uamiName}": {}
      }
    }
  }'
```

# [Azure portal](#tab/portal)

1. In the Azure portal, search for **Microsoft Discovery**.
2. Select **Create bookshelf**.
3. Fill in the required fields:
   - **Resource group** and **Name**
   - **Region** - must match your UAMI region
   - **Search subnet** - dedicated subnet for search services
   - **Private endpoint subnet** - dedicated subnet for managed resource private endpoints
   - **Workload identity** - select your UAMI
4. Under **Tags**, add:
   - `networkIsolation` = `true`
   - `SkipAssociateKeyVaultToNsp` = `true`
5. Select **Review + create**, then **Create**.

---

## Create a bookshelf without network isolation

For development or testing scenarios, you can create a bookshelf without network isolation:

```azurecli
az rest --method PUT \
  --uri "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Discovery/bookshelves/{bookshelfName}?api-version=2026-02-01-preview" \
  --body '{
    "location": "{region}",
    "properties": {
      "workloadIdentities": {
        "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.ManagedIdentity/userAssignedIdentities/{uamiName}": {}
      }
    }
  }'
```

> [!TIP]
> For production workloads, always use network isolation to protect your managed resources with Network Security Perimeter (NSP) and private endpoints.

## Required properties

| Property | Required | Description |
|----------|----------|-------------|
| `location` | Yes | Azure region for the bookshelf. Must match the UAMI region. |
| `workloadIdentities` | Yes | Map of UAMI resource IDs to empty objects. At least one UAMI is required. |
| `searchSubnetId` | For NI | Subnet for search services. Must be in the same virtual network as the workspace. |
| `privateEndpointSubnetId` | For NI | Subnet for managed resource private endpoints. |
| `networkIsolation` tag | For NI | Set to `true` to enable network hardening. |
| `SkipAssociateKeyVaultToNsp` tag | For NI | Set to `true` for Microsoft internal subscriptions only, to avoid conflicts with existing NSP associations. External customer subscriptions don't need this tag. |

## Verify bookshelf creation

Check the provisioning state of your bookshelf:

```azurecli
az rest --method GET \
  --uri "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Discovery/bookshelves/{bookshelfName}?api-version=2026-02-01-preview" \
  --query "{name:name, state:properties.provisioningState}"
```

The bookshelf is ready when `provisioningState` shows `Succeeded`.

## Create a private endpoint for the bookshelf

After the bookshelf is created, you can create a private endpoint to route data-plane API traffic through Azure Private Link:

```azurecli
az network private-endpoint create \
  --resource-group {resourceGroupName} \
  --name {privateEndpointName} \
  --vnet-name {vnetName} \
  --subnet {peSubnet} \
  --private-connection-resource-id "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Discovery/bookshelves/{bookshelfName}" \
  --group-id bookshelf \
  --connection-name {connectionName}
```

Then configure private DNS. See [Configure private DNS](how-to-configure-network-security.md#step-2-configure-private-dns) for details.

| Resource | Private DNS zone |
|----------|-----------------|
| Bookshelf | `privatelink.bookshelf.discovery.azure.com` |

## Subnet planning

When deploying a bookshelf alongside a workspace, plan your virtual network subnets:

| Subnet | Recommended CIDR | Purpose |
|--------|------------------|---------|
| Search subnet | `/24` (256 IPs) | Bookshelf search services |
| PE subnet (shared with workspace) | `/24` (256 IPs) | Private endpoints for managed resources |

> [!IMPORTANT]
> Each Discovery resource (workspace, bookshelf, supercomputer) requires its own unique, non-overlapping subnets. Subnets can't be shared across different Discovery resource instances.

> [!NOTE]
> Each bookshelf requires its own search subnet. The private endpoint subnet can be shared with the workspace if it has sufficient address space.

## Next steps

- [Create a workspace](how-to-create-workspace.md)
- [Configure network security](how-to-configure-network-security.md)
