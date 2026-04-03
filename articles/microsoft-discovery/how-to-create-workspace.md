---
title: Create a Microsoft Discovery workspace
description: Learn how to create a workspace, project, and chat model deployment in Microsoft Discovery.
ms.service: azure
ms.topic: how-to
ms.date: 03/30/2026
ms.author: umamm
author: umamm
---

# Create a Microsoft Discovery workspace

A Microsoft Discovery workspace is the top-level resource for AI-powered scientific research. It provides a unified environment for AI agent execution, tool orchestration, knowledge management, and high-performance computing. When you create a workspace, Azure provisions a managed resource group containing the supporting infrastructure your workloads need, including database, storage, AI services, and key vault resources.

This guide walks you through creating a workspace, project, and chat model deployment.

## Prerequisites

Before you begin, ensure you have completed the [quickstart prerequisites](quickstart-infrastructure-portal.md#prerequisites), including:

- An Azure subscription with the **Microsoft.Discovery** resource provider registered.
- Azure CLI version 2.50 or later.
- A resource group in a [supported region](quickstart-infrastructure-portal.md#prerequisites).
- A user-assigned managed identity (UAMI) for workload authentication:

  ```azurecli
  az identity create \
    --name "{uamiName}" \
    --resource-group "{resourceGroupName}" \
    --location "{region}"
  ```

## Create a workspace

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

1. Sign in to the [Azure portal](https://portal.azure.com).
2. In the search bar, type **Microsoft Discovery** and select it from the results.
3. Select **+ Create** > **Workspace**.
4. On the **Basics** tab:
   - Select your **Subscription** and **Resource group**.
   - Enter a **Workspace name** (must be globally unique, 3–24 characters, lowercase alphanumeric and hyphens only).
   - Select a **Region**.
5. On the **Identity** tab:
   - Select the **User-assigned managed identity** to use for workload authentication.
6. On the **Tags** tab, add any custom tags for your organization.
7. Select **Review + create**, review the configuration summary, then select **Create**.

---

> [!NOTE]
> Workspace provisioning typically takes 10–20 minutes. The operation creates a managed resource group and deploys supporting infrastructure on your behalf.

### Configure private endpoints and DNS

After your workspace is provisioned, you can configure private endpoints to route data-plane API traffic through the Azure backbone instead of the public internet. For detailed steps including private endpoint creation, DNS configuration, and network hardening, see [Configure network security](how-to-configure-network-security.md).

For a complete end-to-end network-isolated deployment, see [End-to-end network-hardened deployment](how-to-deploy-network-hardened-stack.md).

### Verify workspace provisioning

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

### Access the workspace

After provisioning completes, you can access your workspace through:

- **Data-plane API**: `https://{workspaceName}.workspace.discovery.azure.com`
- **Discovery Studio UI**: `https://studio.discovery.microsoft.com/workspaces/{workspaceName}`

> [!TIP]
> Bookmark the Discovery Studio URL for quick access to the workspace UI, where you can manage projects, run experiments, and monitor workloads.

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

## Next steps

- [Configure network security](how-to-configure-network-security.md) — Set up network hardening and private endpoints for your workspace.
- [End-to-end network-hardened deployment](how-to-deploy-network-hardened-stack.md) — Deploy a fully network-isolated Discovery stack.
- [Create a bookshelf](how-to-create-bookshelf.md) — Set up knowledge management with document indexing and retrieval.
- [Manage supercomputers](how-to-manage-supercomputers.md) — Create and manage high-performance computing resources.
