---
title: Enable Zone Redundancy for Workflows
description: Set up availability zones for business continuity and disaster recovery for your logic app workflows by using zone redundancy.
services: azure-logic-apps
ms.suite: integration
ms.reviewers: estfan, shahparth, laveeshb, azla
ms.topic: how-to
ai.usage: ai-assisted
ms.update-cycle: 1095-days
ms.date: 04/13/2026
ms.custom: references_regions
#Customer intent: As an integration developer who works with Azure Logic Apps, I want to enable business continuity and disaster recovery for my logic app workflows by setting up availability zones with zone redundancy.
---

# Enable zone redundancy for logic app workflows by setting up availability zones

[!INCLUDE [logic-apps-sku-consumption-standard](../../includes/logic-apps-sku-consumption-standard.md)]

[!INCLUDE [reliability-az-description](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

For scenarios where you need high reliability for your logic app workflows, you can set up *zone redundancy* with *availability zones* within an Azure region. Azure Logic Apps can then distribute logic app workloads across all the availability zones within a region. This capability protects your apps and their information from datacenter failures within a region.

This guide shows how to enable zone redundancy for your logic apps.

## Prerequisites

- An Azure account and subscription. [Sign up for a free Azure account](https://azure.microsoft.com/pricing/purchase-options/azure-account?cid=msft_learn).

- Make sure that you understand zone redundancy support. When you create your logic app resource, ensure you meet the requirements to use availability zones, including being in a supported region. For more information, see [Reliability in Azure Logic Apps](/azure/reliability/reliability-logic-apps#resilience-to-availability-zone-failures).

- If you have a firewall or restricted environment, you must allow traffic through all the IP addresses required by Azure Logic Apps, Azure-hosted managed connectors, and any custom connectors in the Azure region where you create your logic app workflows. New and existing IP addresses that support availability zone redundancy are published for Azure Logic Apps, managed connectors, and custom connectors.

  For more information, see:

  - [Firewall configuration: IP addresses and service tags](logic-apps-limits-and-config.md#firewall-ip-configuration)
  - [Inbound IP addresses for Azure Logic Apps](logic-apps-limits-and-config.md#inbound)
  - [Outbound IP addresses for Azure Logic Apps](logic-apps-limits-and-config.md#outbound)
  - [Outbound IP addresses for managed connectors and custom connectors](/connectors/common/outbound-ip-addresses)

## Limitations

With HTTP-based actions, certificates exported or created with AES256 encryption don't work for authenticating client certificates. The same certificates also don't work for OAuth authentication.

## Set up zone redundancy for your logic app

For Consumption logic apps, zone redundancy is automatically enabled. You don't need to take any more steps to enable zone redundancy in this type of logic app.

For Standard logic apps only, follow these steps:

1. In the [Azure portal](https://portal.azure.com), start creating a logic app.

   For a tutorial, see [Create an example Standard logic app workflow using the Azure portal](create-single-tenant-workflows-azure-portal.md).

1. On the **Create Logic App** page, select **Workflow Service Plan** or **App Service Environment V3**, based on the hosting option you want to use, and then choose **Select**.

   :::image type="content" source="media/set-up-zone-redundancy-availability-zones/select-standard-plan.png" alt-text="Screenshot shows Azure portal, Create Logic App page, Standard plan types." lightbox="media/set-up-zone-redundancy-availability-zones/select-standard-plan.png":::

1. Under **Zone redundancy**, select **Enabled**.

   :::image type="content" source="media/set-up-zone-redundancy-availability-zones/enable-zone-redundancy-standard.png" alt-text="Screenshot that shows the Azure portal, Create Logic App page with Standard logic app details, and the Enabled option selected under Zone redundancy." lightbox="media/set-up-zone-redundancy-availability-zones/enable-zone-redundancy-standard.png":::

   > [!NOTE]
   >
   > If you select an unsupported Azure region or an existing Windows plan that was created in an unsupported Azure region, the **Zone redundancy** options appear unavailable. Select a different region or create a new Windows plan.

1. Finish creating your logic app workflow.

1. If you use a firewall, ensure you set up access for traffic through the required IP addresses, according to the [prerequisites requirements](#prerequisites).

## Related content

- [Reliability in Azure Logic Apps](/azure/reliability/reliability-logic-apps)
