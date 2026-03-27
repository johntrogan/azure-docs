---
title: Migrate with tooling - Migrate from Basic, Standard, and Premium tiers to Azure Managed Redis
description: Learn how to use migration tooling to move from Azure Cache for Redis Basic, Standard, and Premium tiers to Azure Managed Redis.
ms.date: 03/26/2026
ms.topic: concept-article
ai-usage: ai-assisted
appliesto:
  - ✅ Azure Cache for Redis
  - ✅ Azure Managed Redis

#customer intent: As a developer with Azure Cache for Redis Basic, Standard, or Premium instances, I want to use migration tooling to simplify my migration to Azure Managed Redis.
---

# Migrate with tooling - Basic, Standard, and Premium tiers to Azure Managed Redis

This article describes how to use migration tooling to move from Azure Cache for Redis Basic, Standard, and Premium tiers to Azure Managed Redis.

[!INCLUDE [Redis migration agent skill](../includes/redis-migration-agent-skill.md)]

 [!IMPORTANT]
> Review the [limitations](migrate-basic-standard-premium-migration-options.md#limitations) carefully before choosing this approach.

Use these steps if you choose migration tooling for Basic, Standard, or Premium caches.

## Step 1: Update deployment scripts and create new Azure Managed Redis instance

1. Once you have identified the appropriate Azure Managed Redis SKU, update your deployment scripts (such as ARM templates, Bicep files, or Terraform configurations) to provision Azure Managed Redis instead of Azure Cache for Redis.
1. Use the [SKU mapping table](migrate-basic-standard-premium-understand.md#choose-the-right-azure-managed-redis-size-and-sku) to select the right size (same size or bigger than the existing cache) and performance tier.
1. Create the instance by following the [Quickstart: Create an Azure Managed Redis Instance](../quickstart-create-managed-redis.md).

## Step 2: Configure Entra ID authentication
1. If you use Microsoft Entra ID authentication, configure required managed identities and permissions on the target Azure Managed Redis instance before migration.

## Step 3: Validate and start migration
1. In the Azure portal overview for your Azure Cache for Redis instance, select **Migrate** button from the top level command bar.
1. In the migration pane, select the existing Azure Managed Redis instance you want to migrate to, then click "Validate" button. This will run validations on your Azure Cache for Redis instance to ensure it is ready for migration. I
1. You may see some warnings about potential differences between your Azure Cache for Redis and Azure Managed Redis instance. Youf there are any differences identified as part of this validation, they could be of type **warning** or **error**. For example, if Azure Cache for Redis instance has persistence enabled but the new Azure Managed Redis does not, then this will be flagged as a warning. If Azure Cache for Redis instance is injected in a virtual network, then this will be an error as this is not supported.
1. After reviewing warnings, you can choose to bypass warnings (if present) and then click Migrate button to begin migration.

## Step 4: During migration
1. During migration, cache status changes to **Migrating**. No other management operations can be performed until migration completes.
1. Your client application will experience a connection blip, similar to maintenance experience. When your client application reconnects, it will connect to Azure Managed Redis instance.

## Step 5: Ensure success
1. After migration completes, validate that your application behaves as expected with the migrated endpoint and delete your legacy Azure Cache for Redis cache. 
1. Delete your Azure Cache for Redis instance. Note that the Azure Cache for Redis hostname will continue to point to the new Azure Managed Redis instance even after the Azure Cache for Redis instance is deleted.

## Step 6: Update client application to use Azure Managed Redis hostname
1. Update applications to use the Azure Managed Redis hostname (`<cachename>.<region>.redis.azure.net`) and retire the unused Azure Cache for Redis endpoint using UnlinkEndpoint API.

## Related content

- [Migrate from Basic, Standard, and Premium tiers to Azure Managed Redis](migrate-basic-standard-premium-overview.md)
- [Understand the differences](migrate-basic-standard-premium-understand.md)
- [Plan migration execution](migrate-basic-standard-premium-self-service.md)
- [Migration options](migrate-basic-standard-premium-options.md)
- [What is Azure Managed Redis?](../overview.md)
