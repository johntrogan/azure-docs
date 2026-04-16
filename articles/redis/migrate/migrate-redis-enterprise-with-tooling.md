---
title: Migrate with tooling - Migrate from Redis Enterprise to Azure Managed Redis
description: Learn how to use migration tooling to move from Azure Cache for Redis Enterprise tier to Azure Managed Redis.
ms.date: 04/14/2026
ms.topic: concept-article
ai-usage: ai-assisted
appliesto:
  - ✅ Azure Cache for Redis Enterprise
  - ✅ Azure Managed Redis

#customer intent: As a developer with Azure Cache for Redis Enterprise instances, I want to use migration tooling to simplify my migration to Azure Managed Redis.
---

# Migrate with tooling (preview) - Redis Enterprise to Azure Managed Redis

Azure provides built-in migration tooling (preview) that automates endpoint migration from your existing Azure Cache for Redis Enterprise instance to a precreated Azure Managed Redis instance. After you create a new Azure Managed Redis instance and initiate the migration, the tooling updates your Azure Cache for Redis Enterprise hostname to point to Azure Managed Redis, so your client applications reconnect automatically to the Azure Managed Redis instance using the same hostname and access key. Once you validate the migration, delete the old Azure Cache for Redis Enterprise instance and update your client applications to use the new Azure Managed Redis hostname.

[!INCLUDE [Redis migration agent skill](../includes/redis-migration-agent-skill.md)]

Use these steps if you choose migration tooling for Enterprise caches using Azure Portal.

## Step 1: Update deployment scripts and create new Azure Managed Redis instance

1. Once you have identified the appropriate Azure Managed Redis SKU, update your deployment scripts (such as ARM templates, Bicep files, or Terraform configurations) to provision Azure Managed Redis instead of Azure Cache for Redis Enterprise.
1. Use the [size guidance](migrate-redis-enterprise-understand.md#choose-the-right-azure-managed-redis-size) to select the right size (same size or bigger than the existing cache) and performance tier.
1. Create the instance by following the [Quickstart: Create an Azure Managed Redis Instance](../quickstart-create-managed-redis.md).

## Step 2a: Configure Entra ID authentication (optional)

1. If you use Microsoft Entra ID authentication, configure required managed identities and permissions on the target Azure Managed Redis instance before migration.

## Step 2b: Data migration (optional)

1. If you need your data to be copied over to your new Azure Managed Redis instance, review multiple [data migration strategies](migrate-redis-enterprise-self-service.md#step-3-migrate-your-data).

## Step 3: Validate and start migration

1. In the Azure portal, use the **Resource** menu for your Azure Cache for Redis Enterprise instance and go to **Overview**.
1. Select **Migrate** from the top level command bar.
1. In the migration pane, select the existing Azure Managed Redis instance you want to migrate to, then select **Validate**. This will run validations on your Azure Cache for Redis Enterprise instance to ensure it is ready for migration.
1. You may see some warnings about potential differences between your Azure Cache for Redis Enterprise and Azure Managed Redis instances. If there are differences identified, they could be of type **warning** or **error**.
1. After reviewing warnings, you can choose to bypass warnings (if present) and then select **Migrate** to begin migration.

## Step 4: During migration

1. During migration, cache status changes to **Migrating**. No other management operations can be performed until migration completes.
1. Your client application will experience a connection blip, similar to maintenance experience. When your client application reconnects, it will connect to Azure Managed Redis instance.

## Step 5: Ensure success and delete old Azure Cache for Redis Enterprise instance

1. After migration completes, validate that your application behaves as expected with the migrated endpoint and delete your legacy Azure Cache for Redis Enterprise cache.
1. Delete your Azure Cache for Redis Enterprise instance. Note that the Azure Cache for Redis Enterprise hostname will continue to point to the new Azure Managed Redis instance even after the Azure Cache for Redis Enterprise instance is deleted.

## Step 6: Update client application to use Azure Managed Redis hostname

1. Update applications to use the Azure Managed Redis hostname (`<cachename>.<region>.redis.azure.net`) and retire the unused Azure Cache for Redis Enterprise hostname.

## Migrate using PowerShell script

You can also use Azure PowerShell commands to perform pre-validation and then initiate migration. You can also check the status of migration operation and cancel it, if required. You can find a PowerShell script for migration [here](https://github.com/AzureManagedRedis/amr-migration-scripts).

## Related content

- [Migrate from Redis Enterprise to Azure Managed Redis](migrate-redis-enterprise-overview.md)
- [Understand the differences](migrate-redis-enterprise-understand.md)
- [Plan migration execution](migrate-redis-enterprise-self-service.md)
- [What is Azure Managed Redis?](../overview.md)
