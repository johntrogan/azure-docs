---
title: Azure Event Hubs with confidential computing
description: Learn how to enable confidential computing on Azure Event Hubs Dedicated namespaces to protect data in use with hardware-based trusted execution environments.
ms.topic: conceptual
ms.date: 01/22/2026
ms.service: azure-event-hubs
author: axisc
ms.author: achhabria
# Customer intent: As a security administrator or developer, I want to enable confidential computing on Azure Event Hubs to protect sensitive event data in use with hardware-based isolation.
---

# Azure Event Hubs confidential computing overview

Azure Event Hubs Dedicated supports [confidential computing](../confidential-computing/overview.md) to protect your event data in use. Confidential computing uses hardware-based trusted execution environments (TEEs) to provide enhanced data protection, preventing unauthorized access to your events while they're being processed.

When you enable confidential computing on an Event Hubs Dedicated namespace, your data benefits from hardware-level isolation in addition to existing encryption at rest and in transit. This capability helps organizations that handle sensitive or regulated data meet strict security and compliance requirements.

## Benefits

Confidential computing for Azure Event Hubs provides the following advantages:

- **No code changes required**: Enable confidential computing at the namespace level without modifying your applications or event processing patterns.
- **Defense in depth**: Combines with existing Event Hubs security features like [customer-managed keys](configure-customer-managed-key.md), [private endpoints](private-link-service.md), and [managed identities](authenticate-managed-identity.md).
- **Event streaming protection**: Your event hubs benefit from hardware-level isolation during event processing.

## Regional availability

Confidential computing for Azure Event Hubs is available in select regions.

| Region |
|--------|
| Korea Central |
| UAE North |

## Limitations

The following limitations apply to confidential computing for Azure Event Hubs:

- Confidential computing is available only on the **[Dedicated tier](event-hubs-dedicated-overview.md)**.
- You must enable confidential computing during namespace creation. You can't enable it on existing namespaces.

## Enable confidential computing by using the Azure portal

1. Go to the [Azure portal](https://portal.azure.com) and open the Event Hubs namespace creation page.

1. Select **Dedicated** for the pricing tier.

1. Select a [supported region](#regional-availability) as the location.

1. For **Confidential compute**, select **Enabled**.

    :::image type="content" source="./media/confidential-computing/enable-confidential-computing-portal.png" alt-text="Screenshot showing the Create namespace page with the Confidential compute toggle enabled.":::

1. Fill in the remaining required fields for your namespace configuration.

1. Select **Review + create**, and then select **Create** to deploy the namespace with confidential computing enabled.

## Enable confidential computing by using a template

You can enable confidential computing programmatically by including the `platformCapabilities` property in your deployment template.

# [Bicep](#tab/bicep)

The following Bicep file creates an Event Hubs Dedicated namespace with confidential computing enabled:

```bicep
@description('Name of the Event Hubs namespace')
param namespaceName string

@description('Location for the namespace. Must be a region that supports confidential computing.')
@allowed([
  'koreacentral'
  'uaenorth'
])
param location string = 'uaenorth'

resource eventHubNamespace 'Microsoft.EventHub/namespaces@2025-05-01-preview' = {
  name: namespaceName
  location: location
  sku: {
    name: 'Premium'
    tier: 'Premium'
    capacity: 1
  }
  kind: 'ClusterlessNamespace'
  properties: {
    platformCapabilities: {
      confidentialCompute: {
        mode: 'Enabled'
      }
    }
  }
}
```

# [ARM template](#tab/arm)

The following ARM template creates an Event Hubs Dedicated namespace with confidential computing enabled:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "namespaceName": {
            "type": "string",
            "metadata": {
                "description": "Name of the Event Hubs namespace"
            }
        },
        "location": {
            "type": "string",
            "defaultValue": "uaenorth",
            "allowedValues": [
                "koreacentral",
                "uaenorth"
            ],
            "metadata": {
                "description": "Location for the namespace. Must be a region that supports confidential computing."
            }
        }
    },
    "resources": [
        {
            "type": "Microsoft.EventHub/namespaces",
            "apiVersion": "2025-05-01-preview",
            "name": "[parameters('namespaceName')]",
            "location": "[parameters('location')]",
            "kind": "ClusterlessNamespace",
            "sku": {
                "name": "Premium",
                "tier": "Premium",
                "capacity": 1
            },
            "properties": {
                "platformCapabilities": {
                    "confidentialCompute": {
                        "mode": "Enabled"
                    }
                }
            }
        }
    ]
}
```

---

## Combine confidential computing with customer-managed keys

For maximum data protection, combine confidential computing with [customer-managed keys](configure-customer-managed-key.md) backed by [Azure Key Vault Managed HSM](/azure/key-vault/managed-hsm/overview). This combination ensures that:

- Your data is protected in use by confidential computing.
- Your encryption keys are stored in validated hardware security modules.
- You maintain full control over your encryption keys.

## Use Azure Policy to enforce confidential computing

Create an Azure Policy definition to enforce that all Dedicated Event Hubs namespaces in your organization have both confidential computing and customer-managed keys enabled. This approach ensures consistent security configuration across your Azure environment.

The following policy definition denies or audits the creation of Dedicated Event Hubs namespaces that don't meet these security requirements:

```json
{
    "mode": "All",
    "parameters": {
        "effect": {
            "type": "String",
            "metadata": {
                "displayName": "Effect",
                "description": "Deny or Audit"
            },
            "allowedValues": [
                "Deny",
                "Audit"
            ],
            "defaultValue": "Deny"
        }
    },
    "policyRule": {
        "if": {
            "allOf": [
                {
                    "field": "type",
                    "equals": "Microsoft.EventHub/namespaces"
                },
                {
                    "field": "Microsoft.EventHub/namespaces/sku.tier",
                    "equals": "Premium"
                },
                {
                    "field": "Microsoft.EventHub/namespaces/kind",
                    "equals": "ClusterlessNamespace"
                },
                {
                    "anyOf": [
                        {
                            "anyOf": [
                                {
                                    "not": {
                                        "field": "Microsoft.EventHub/namespaces/encryption.keySource",
                                        "equals": "Microsoft.KeyVault"
                                    }
                                },
                                {
                                    "not": {
                                        "field": "Microsoft.EventHub/namespaces/encryption.keyVaultProperties[*].keyVaultUri",
                                        "contains": ".managedhsm.azure.net/"
                                    }
                                },
                                {
                                    "anyOf": [
                                        {
                                            "field": "identity.type",
                                            "equals": "None"
                                        },
                                        {
                                            "field": "identity.type",
                                            "exists": false
                                        }
                                    ]
                                }
                            ]
                        },
                        {
                            "not": {
                                "field": "Microsoft.EventHub/namespaces/platformCapabilities.confidentialCompute.mode",
                                "equals": "Enabled"
                            }
                        }
                    ]
                }
            ]
        },
        "then": {
            "effect": "[parameters('effect')]"
        }
    }
}
```

To use this policy, create a custom policy definition in Azure Policy and assign it to the appropriate scope, such as a management group, subscription, or resource group.

> [!NOTE]
> When combining confidential computing with customer-managed keys, use a user-assigned managed identity. This requirement exists because the identity must be granted access to the Managed HSM before creating the namespace. A system-assigned identity only exists after the namespace is created.

## Related content

- [What is confidential computing?](../confidential-computing/overview.md)
- [Azure confidential computing products](../confidential-computing/overview-azure-products.md)
- [Confidential computing use cases](../confidential-computing/use-cases-scenarios.md)
- [Configure customer-managed keys for Azure Event Hubs](configure-customer-managed-key.md)
- [Azure Event Hubs Dedicated](event-hubs-dedicated-overview.md)
- [Azure Key Vault Managed HSM](/azure/key-vault/managed-hsm/overview)
