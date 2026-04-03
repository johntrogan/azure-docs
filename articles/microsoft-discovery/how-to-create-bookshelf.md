---
title: Create a Microsoft Discovery bookshelf
description: Learn how to create and configure a Microsoft Discovery bookshelf for knowledge management with data ingestion, indexing, and retrieval capabilities.
ms.service: azure
ms.topic: how-to
ms.date: 03/30/2026
ms.author: umamm
author: umamm
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

> [!IMPORTANT]
> The UAMI must be in the **same Azure region** as the bookshelf. A location mismatch causes the creation to fail.

## Create a bookshelf

# [Azure CLI](#tab/azure-cli)

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

# [Azure portal](#tab/portal)

1. In the Azure portal, search for **Microsoft Discovery**.
2. Select **Create bookshelf**.
3. Fill in the required fields:
   - **Resource group** and **Name**
   - **Region** - must match your UAMI region
   - **Workload identity** - select your UAMI
4. Select **Review + create**, then **Create**.

---

## Required properties

| Property | Required | Description |
|----------|----------|-------------|
| `location` | Yes | Azure region for the bookshelf. Must match the UAMI region. |
| `workloadIdentities` | Yes | Map of UAMI resource IDs to empty objects. At least one UAMI is required. |

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

## Next steps

- [Manage workspaces](how-to-manage-workspaces.md)
- [Configure network security](how-to-configure-network-security.md) — Add network isolation to protect your bookshelf with NSP and private endpoints.
- [End-to-end network-hardened deployment](how-to-deploy-network-hardened-stack.md) — Deploy a fully network-isolated Discovery stack including bookshelves.
