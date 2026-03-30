---
title: Enable Redis Keyspace Notifications in Azure Managed Redis
description: Enable Redis keyspace notifications in Azure Managed Redis so clients can monitor changes to cache keys and values in real time.
author: dlepow
ms.author: danlep
ms.date: 03/30/2026
ms.topic: how-to
ms.service: azure-managed-redis
---

# Enable Redis keyspace notifications

Redis keyspace notifications let a subscriber client receive events when commands affect keys. Use keyspace notifications to monitor changes to keys and values in your Redis cache. 

This article shows how to deploy a cache with keyspace notifications enabled, connect Redis CLI clients, subscribe to notification channels, and test the resulting events.

## Prerequisites

- `redis-cli` command line tool. For installation steps, see [Use client tools to manage data in Azure Managed Redis](how-to-redis-access-data.md).
- Understanding of Azure Resource Manager (ARM) templates. For more information, see [Azure Resource Manager documentation](/azure/azure-resource-manager/management/overview).
- Azure CLI

## Deploy a cache with keyspace notifications enabled

<!-- Could this be modified to update an existing cache instead? -->

In this example, use an Azure Resource Manager (ARM) template to deploy a Redis cache with keyspace notifications enabled.

Modify the `CacheName` and `Region` parameters in the following template, and save the file as `KeyspaceTemplate.json`:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "cachename": {
            "defaultValue": "{CacheName}",
            "type": "String"
        },
        "region": {
            "defaultValue": "{Region}",
            "type": "String"
        }
    },
    "variables": {},
    "resources": [
        {
            "type": "Microsoft.Cache/redisEnterprise",
            "apiVersion": "2026-02-01-preview",
            "name": "[parameters('cachename')]",
            "location": "[parameters('region')]",
            "sku": {
                "name": "Balanced_B5"
            },
            "identity": {
                "type": "None"
            },
            "properties": {
                "minimumTlsVersion": "1.2",
                "publicNetworkAccess": "Enabled"
            }
        },
        {
            "type": "Microsoft.Cache/redisEnterprise/databases",
            "apiVersion": "2026-02-01-preview",
            "name": "[concat(parameters('cachename'), '/default')]",
            "dependsOn": [
                "[resourceId('Microsoft.Cache/redisEnterprise', parameters('cachename'))]"
            ],
            "properties": {
                "clientProtocol": "Encrypted",
                "port": 10000,
                "clusteringPolicy": "OSSCluster",
                "evictionPolicy": "NoEviction",
                "persistence": {
                    "aofEnabled": false,
                    "rdbEnabled": false
                },
                "notifyKeyspaceEvents": "KEA"
            }
        }
    ]
}
```

Deploy the template using the [az deployment group create](/azure/deployment/group#az_deployment_group_create) Azure CLI command. In the following example, the deployment is in the *exampleRG* resource group.

```azurecli
az deployment group create --resource-group exampleRG --template-file KeyspaceTemplate.json
```

## Enable access key authentication

After the cache is created, enable access keys in the Azure portal, and copy the primary key for use with the Redis CLI.

1. Go to your cache in the [Azure portal](https://portal.azure.com/).
1. Go to **Settings** > **Authentication**.
1. Select the **Access keys** tab.
1. Set **Access Keys Authentication** to **Enabled**.
1. Copy the **Primary key**.

## Connect Redis CLI clients

Open two terminals and connect both clients to the cache.

```redis
redis-cli -h {yourcachename}.{region}.redis.azure.net -p 10000 -a {YourAccessKey} --tls -c
```

Use one terminal as the subscriber that receives keyspace notifications and the other as the operator that runs Redis commands.

## Verify and subscribe to events

In either terminal, confirm that the notification string is configured.

```redis
CONFIG GET notify-keyspace-events
```

In the subscriber terminal, choose the subscription pattern that matches the events you want to observe. The following are examples:

- Subscribe to all events and all keys:

    ```redis
    PSUBSCRIBE __keyevent@0__:* __keyspace@0__:*
    ```

- Subscribe to a specific event type:

    ```redis
    SUBSCRIBE __keyevent@0__:set
    ```

- Subscribe to a specific key:

    ```redis
    SUBSCRIBE __keyspace@0__:mykey
    ```

:::image type="content" source="media/enable-redis-keyspace-notifications/keyspace-notification-redis-cli.png" alt-text="Redis CLI session showing keyspace notification subscription output." lightbox="media/enable-redis-keyspace-notifications/keyspace-notification-redis-cli.png":::

## Test notifications

Switch to the operator terminal and run a few Redis commands against a test key.

```redis
SET mykey "hello"
EXPIRE mykey 10
DEL mykey
```

The subscriber terminal shows which events occurred and which keys were affected.

## Update notification settings

You can change the keyspace notification configuration later without recreating the cache.

1. Update the `notifyKeyspaceEvents` value in `KeyspaceTemplate.json` while keeping `CacheName` and `Region` the same.
1. Redeploy the template.

```bash
az deployment group create --resource-group exampleRG --template-file KeyspaceTemplate.json
```

Redeploying updates the notification settings without data loss, which lets you enable, disable, or change the events that are tracked, such as moving from `KEA` to `Ex` to track expiration events only.

## Related content

- [Keyspace notifications in Azure Cache for Redis](/azure/azure-cache-for-redis/cache-configure#keyspace-notifications-advanced-settings)
- [Redis keyspace notifications documentation](https://redis.io/docs/latest/develop/pubsub/keyspace-notifications/)
