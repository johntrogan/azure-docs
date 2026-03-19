---
title: Upgrade your Redis Version in Azure Managed Redis
description: Learn how to upgrade the version of Azure Managed Redis.
ms.date: 05/18/2025
ms.topic: how-to
ms.custom:
  - ignite-2024
  - build-2025
appliesto:
  - ✅ Azure Managed Redis
---

# Upgrade the version of your Azure Managed Redis instance

New versions of Redis server software are frequently released with new features, more commands, and stability improvements. By maintaining your Redis instances with the latest version of Redis, you can ensure that you get the best possible Redis experience.

This article details how to upgrade your Redis instance to the latest version available in Azure Managed Redis.

> [!IMPORTANT]
> Following the [standard Redis versioning](https://redis.io/about/releases/), this article only covers upgrades to the _major_ version of Redis, not the _minor_ or _patch_ versions. Updates to the minor and patch versions are made automatically during the normal patching cycle each month.
>

### Current versions

These Redis versions are available for each tier.

| Tier                         |        Available Redis version       |
|:------------------------------------------------- |:------------------------------------:|
| Memory Optimized, Balanced, Compute Optimized     |   Redis 7.4 (Preview) |
| Flash Optimized | Redis 7.4 (preview)  |

## Upgrade options

You can choose from automatic or manual upgrades. Automatic upgrades are part of the standard patching process. You can use the manual process to start upgrades that are available outside the normal automatic process.

### Automatic upgrades

Redis server version upgrades are made automatically as a part of the standard monthly patching process. Upgrades to the latest version of Redis occur when that Redis version reaches general availability (GA) on Azure.

When a new version becomes generally available, your Redis instance is automatically upgraded to the new version unless you defer it ahead of time. For more information on deferring an upgrade, see [Defer upgrades](#defer-upgrades).

### Manual upgrades

You can also choose to manually upgrade to the latest Redis version. Manual upgrades provide two benefits:

- You control when the upgrade occurs.
- You can upgrade to preview releases of Redis server.

> [!WARNING]
> After you upgrade your Redis instance, you can't downgrade it to the previous version.
>

1. In the portal, use the Resource menu to go to the cache **Overview**. On the working pane, select **Upgrade**.

   :::image type="content" source="media/how-to-upgrade/managed-redis-upgrade-overview.png" alt-text="Screenshot that shows the Upgrade pane, the current version, and the available version." :::

1. You see an **Upgrade Redis** pane that displays the current Redis version, and the versions that you can upgrade to. To confirm and begin the upgrade process, select **Start Upgrade**.

   :::image type="content" source="media/how-to-upgrade/managed-redis-upgrade-pane.png" alt-text="Screenshot that shows Overview on the Resource menu and the Upgrade Redis pane.":::

   If you're already running the latest version of Redis software available, the **Upgrade** button is disabled.

### Upgrade deferment

You can defer an automatic upgrade to a new version of Redis software by up to 90 days. This option gives you time to test new versions and ensure that everything works as expected. The cache is upgraded either 90 days after the new Redis version reaches GA, or whenever you manually trigger the upgrade.

The deferral option must be selected before a new Redis version reaches GA for it to take effect before the automatic upgrade occurs.

To defer upgrades to your cache, go to **Advanced Settings** on the **Resource** menu, and select the **Defer Redis DB version updates** box.

:::image type="content" source="media/how-to-upgrade/managed-redis-defer-upgrade.png" alt-text="Screenshot that shows Advanced settings selected on the Resource menu and a red box around the text Defer Redis DB version updates.":::

> [!IMPORTANT]
> When you select the option to defer upgrades, your selection only applies to the next automatic upgrade event. You can't use the defer button to downgrade caches that were already upgraded.

## Pre-upgrade considerations

We intend for each new Redis version to be a seamless upgrade from previous versions designed for backwards compatibility. However, small changes and bug fixes do occur, which can cause application changes. You should keep these changes in mind.

### Client version

When you use an outdated Redis client, new commands or Redis features aren't properly supported. We always recommend that you update to the latest stable version of your Redis client, as newer versions often have stability and performance improvements. For more information on configuring your client library, see [Best practices using client libraries](best-practices-client-libraries.md).

### Breaking changes

Each version of Redis often has a few minor bug fixes that can present breaking changes. If you have concerns, we recommend reviewing the Redis 7.4 release notes before you upgrade your Redis version:

- [Redis 7.4 release notes](https://raw.githubusercontent.com/redis/redis/7.4/00-RELEASENOTES)
