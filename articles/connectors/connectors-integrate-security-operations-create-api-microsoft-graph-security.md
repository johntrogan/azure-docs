---
title: Improve Security and Threat Detection for Automated Workflows
description: Improve threat protection, detection, and responses for Consumption workflows in Azure Logic Apps by using Microsoft Graph Security.
services: azure-logic-apps
ms.suite: integration
author: ecfan
ms.author: preetikr
ms.reviewers: estfan, azla
ms.topic: how-to
ms.update-cycle: 365-days
ms.date: 04/02/2026
# Customer intent: As an integration developer who works with Azure Logic Apps, I want to improve threat protection, detection, and responses for my Consumption workflows by using Microsoft Graph Security.
---

# Improve threat protection, detection, and response for workflows in Azure Logic Apps by using Microsoft Graph Security

[!INCLUDE [logic-apps-sku-consumption](~/reusable-content/ce-skilling/azure/includes/logic-apps-sku-consumption.md)]

By using [Azure Logic Apps](../logic-apps/logic-apps-overview.md) with the [Microsoft Graph Security](/graph/security-concept-overview) connector, you can improve how your app detects, protects, and responds to threats by creating automated workflows for integrating Microsoft security products, services, and partners. For example, you can create [Microsoft Defender for Cloud playbooks](../security-center/workflow-automation.yml) that monitor and manage Microsoft Graph Security entities, such as alerts. Here are some scenarios that the Microsoft Graph Security connector supports:

- Get alerts based on queries or by alert ID. For example, get a list that includes high severity alerts.

- Update alerts. For example, update alert assignments, add comments to alerts, or tag alerts.

- Monitor when alerts are created or changed by creating [alert subscriptions (webhooks)](/graph/api/resources/change-notifications-api-overview).

- Manage your alert subscriptions. For example, get active subscriptions, extend the expiration time for a subscription, or delete a subscription.

Your logic app's workflow can use actions that get responses from the Microsoft Graph Security connector and make that output available to other actions in your workflow. You can also have other actions in your workflow use the output from the Microsoft Graph Security connector actions. For example, if you get high severity alerts through the Microsoft Graph Security connector, you can send those alerts in an email message by using the Outlook connector.

## Prerequisites

- An Azure account and subscription. [Get a free Azure account](https://azure.microsoft.com/pricing/purchase-options/azure-account?cid=msft_learn).

- To use the Microsoft Graph Security connector, you must *explicitly give* the Microsoft Entra tenant administrator consent, which is part of the [Microsoft Graph Security Authentication requirements](/graph/security-authorization). This consent requires the Microsoft Graph Security connector's application ID and name, which you can also find in the [Azure portal](https://portal.azure.com):

  | Property | Value |
  | -------------------- | --------------------------------- |
  | **Application Name** | `MicrosoftGraphSecurityConnector` |
  | **Application ID** | `c4829704-0edc-4c3d-a347-7c4a67586f3c` |

  To grant consent for the connector, do one of the following actions:

  - [Request tenant administrator consent be granted for Microsoft Entra applications](../active-directory/develop/v2-permissions-and-consent.md).

  - During your logic app's first run, have your app request consent from your Microsoft Entra tenant administrator through the [application consent experience](../active-directory/develop/application-consent-experience.md).

- Basic knowledge of how to create logic apps.

- A logic app in which you want to access Microsoft Graph Security entities, such as alerts. To use a Microsoft Graph Security trigger, you need a blank logic app. To use a Microsoft Graph Security action, you need a logic app that starts with the appropriate trigger for your scenario.

## Connect to Microsoft Graph Security

[!INCLUDE [Create connection general intro](../../includes/connectors-create-connection-general-intro.md)]

1. Sign in to the [Azure portal](https://portal.azure.com/), and open your logic app in Logic App Designer.

1. Based on your logic app, do one of these steps:

   - For blank logic apps, add the trigger and any other actions that you want before you add a Microsoft Graph Security action.

   - For existing logic apps, under the last step where you want to add a Microsoft Graph Security action, select **New step**.

1. To add an action between steps, move your pointer over the arrow between steps. Select the plus sign (+) that appears, and select **Add an action**.

1. In the **Add an action** search box, enter **Microsoft Graph Security** as your filter. From the actions list, select the action you want.

1. Sign in with your Microsoft Graph Security credentials.

1. Provide the necessary details for your selected action and continue building your logic app's workflow.

## Add triggers

In Azure Logic Apps, every logic app must start with a [trigger](../logic-apps/logic-apps-overview.md#logic-app-concepts), which fires when a specific event happens or when a specific condition is met. Each time that the trigger fires, the logic apps engine creates a logic app instance and starts running your app's workflow.

> [!NOTE]
> When a trigger fires, the trigger processes all the new alerts. If no alerts are received, the trigger run is skipped. The next trigger poll happens based on the recurrence interval that you specify in the trigger's properties.

The following example shows how you can start a logic app workflow when new alerts are sent to your app:

1. In the Azure portal, create a blank logic app, which opens the Logic App Designer.

1. Select **Add a trigger** on the designer canvas. In the **Add a trigger** search box, enter **Microsoft Graph Security** as your filter. From the triggers list, select this trigger: **On all new alerts**.

1. In the trigger, provide information about the alerts that you want to monitor.

   | Property | Property (JSON) | Required | Type | Description |
   | ------------ | ---------- | --- | ------- | -------------- |
   | **Interval** | `interval` | Yes | Integer | A positive integer that describes how often the workflow runs, based on the frequency unit of time. Allowable intervals: <p><p>- **Month**: 1-16 months<br>- **Week**: 1-71 weeks<br>- **Day**: 1-500 days<br>- **Hour**: 1-12,000 hours<br>- **Minute**: 1-72,000 minutes<br>- **Second**: 1-9,999,999 seconds<p>For example, if the interval is 6, and the frequency is **Month**, then the recurrence is every 6 months. |
   | **Frequency** | `frequency` | Yes | String | The unit of time for the recurrence: **Month**, **Week**, **Day**, **Hour**, **Minute**, or **Second**. |
   | **Time zone** | `timeZone` | No | String | Select the time zone that you want to apply. Because this trigger doesn't accept [UTC offset](https://en.wikipedia.org/wiki/UTC_offset), it applies only when you specify a start time. |
   | **Start time** | `startTime` | No | String | Provide a start date and time in this format: <p><p>- If you select a time zone: YYYY-MM-DDThh:mm:ss<p>- If you don't select a time zone: YYYY-MM-DDThh:mm:ssZ<p>For example, for September 18, 2017 at 2:00 PM, enter **2017-09-18T14:00:00** and select a time zone such as **(UTC-08:00) Pacific Time (US & Canada)**. Or, enter **2017-09-18T14:00:00Z** without specifying a time zone. <p>The start time has a maximum of 49 years in the future and must follow the [ISO 8601 date time specification](https://en.wikipedia.org/wiki/ISO_8601#Combined_date_and_time_representations) in [UTC date time format](https://en.wikipedia.org/wiki/Coordinated_Universal_Time), but without a [UTC offset](https://en.wikipedia.org/wiki/UTC_offset). If you don't select a time zone, you must add the letter **Z** at the end of the specified time without any spaces. **Z** refers to the equivalent [nautical time](https://en.wikipedia.org/wiki/Nautical_time). <p>For simple schedules, the start time is the first occurrence. For complex schedules, the trigger doesn't fire any sooner than the start time. For more information, see [Patterns for start date and time](../logic-apps/concepts-schedule-automated-recurring-tasks-workflows.md#start-time). |

1. When you're done, on the designer toolbar, select **Save**.

1. Add one or more actions to your logic app for the tasks you want to perform with the trigger results.

## Add actions

With the Microsoft Graph Security connector, you can add actions to manage alerts, alert subscriptions, and threat intelligence indicators.

### Manage alerts

To filter, sort, or get the most recent results, provide *only* the [ODATA query parameters supported by Microsoft Graph](/graph/query-parameters). Don't specify the complete base URL or the HTTP action, for example, `https://graph.microsoft.com/v1.0/security/alerts`, or the `GET` or `PATCH` operation.

The following example shows the parameters for a **Get alerts** action when you want a list with high severity alerts:

`Filter alerts value as Severity eq 'high'`

For more information about the queries you can use with this connector, see the [Microsoft Graph Security alerts reference](/graph/api/alert-list). To build enhanced experiences with this connector, learn about the [schema properties alerts](/graph/api/resources/alert) that the connector supports.

The following table lists supported alert actions.

| Action | Description |
| ------ | ----------- |
| **Get alerts** | Get alerts filtered based on one or more [alert properties](/graph/api/resources/alert), for example, `Provider eq 'Azure Security Center' or 'Palo Alto Networks'`. |
| **Get alert by ID** | Get a specific alert based on the alert ID. |
| **Update alert** | Update a specific alert based on the alert ID. To make sure you pass the required and editable properties in your request, see the [editable properties for alerts](/graph/api/alert-update). For example, to assign an alert to a security analyst to investigate, update the alert's **Assigned to** property. |

### Manage alert subscriptions

Microsoft Graph supports [*subscriptions*](/graph/api/resources/subscription), or [*webhooks*](/graph/api/resources/change-notifications-api-overview). To get, update, or delete subscriptions, provide the [ODATA query parameters supported by Microsoft Graph](/graph/query-parameters) to the Microsoft Graph entity construct and include `security/alerts` followed by the ODATA query.

Don't include the base URL, for example, `https://graph.microsoft.com/v1.0`. Instead, use the format in the following example:

`security/alerts?$filter=status eq 'NewAlert'`

The following table lists supported alert subscription actions.

| Action | Description |
| ------ | ----------- |
| **Create subscriptions** | [Create a subscription](/graph/api/subscription-post-subscriptions) that notifies you about any changes. You can filter this subscription for the specific alert types you want. For example, you can create a subscription that notifies you about high severity alerts. |
| **Get active subscriptions** | Get a [subscription list](/graph/api/subscription-list) that shows active and unexpired subscriptions. |
| **Update subscription** | [Update a subscription](/graph/api/subscription-update) by providing the subscription ID. For example, to extend your subscription, you can update the subscription's `expirationDateTime` property. |
| **Delete subscription** | [Delete a subscription](/graph/api/subscription-delete) by providing the subscription ID. |

### Manage threat intelligence indicators

To filter, sort, or get the most recent results, provide *only* the [ODATA query parameters supported by Microsoft Graph](/graph/query-parameters). Don't specify the complete base URL or the HTTP action, for example, `https://graph.microsoft.com/beta/security/tiIndicators`, or the `GET` or `PATCH` operation. The following example shows the parameters for a **Get tiIndicators** action for a list that has the `DDoS` threat type:

`Filter threat intelligence indicator value as threatType eq 'DDoS'`

For more information about the queries that you can use with this connector, see [Optional query parameters](/graph/api/tiindicators-list#optional-query-parameters). To build enhanced experiences with this connector, learn about the [schema properties threat intelligence indicator](/graph/api/resources/tiindicator) that the connector supports.

The following table lists supported threat intelligence indicator actions.

| Action | Description |
| ------ | ----------- |
| **Get threat intelligence indicators** | Get tiIndicators filtered based on one or more [tiIndicator properties](/graph/api/resources/tiindicator), for example, `threatType eq 'MaliciousUrl' or 'DDoS'`. |
| **Get threat intelligence indicator by ID** | Get a specific tiIndicator based on the tiIndicator ID. |
| **Create threat intelligence indicator** | Create a new tiIndicator by posting to the tiIndicators collection. To make sure that you pass the required properties in your request, refer to the [required properties for creating tiIndicator](/graph/api/tiindicators-post). |
| **Submit multiple threat intelligence indicators** | Create multiple new tiIndicators by posting a tiIndicators collection. To make sure that you pass the required properties in your request, refer to the [required properties for submitting multiple tiIndicators](/graph/api/tiindicator-submittiindicators). |
| **Update threat intelligence indicator** | Update a specific tiIndicator based on the tiIndicator ID. To make sure you pass the required and editable properties in your request, see the [editable properties for tiIndicator](/graph/api/tiindicator-update). For example, to update the action to apply if the indicator is matched from within the targetProduct security tool, you can update the tiIndicator's **action** property. |
| **Update multiple threat intelligence indicators** | Update multiple tiIndicators. To make sure you pass the required properties in your request, refer to the [required properties for updating multiple tiIndicators](/graph/api/tiindicator-updatetiindicators). |
| **Delete threat intelligence indicator by ID** | Delete a specific tiIndicator based on the tiIndicator ID. |
| **Delete multiple threat intelligence indicators by IDs** | Delete multiple tiIndicators by their IDs. To make sure that you pass the required properties in your request, refer to the [required properties for deleting multiple tiIndicators by IDs](/graph/api/tiindicator-deletetiindicators). |
| **Delete multiple threat intelligence indicators by external IDs** | Delete multiple tiIndicators by the external IDs. To make sure that you pass the required properties in your request, refer to the [required properties for deleting multiple tiIndicators by external IDs](/graph/api/tiindicator-deletetiindicatorsbyexternalid). |

## Connector reference

For technical details about triggers, actions, and limits, which the connector's OpenAPI specification describes, see the [Microsoft Graph Security connector reference](/connectors/microsoftgraphsecurity/).

## Related content

- [Managed connectors for Azure Logic Apps](/connectors/connector-reference/connector-reference-logicapps-connectors)
- [Built-in connectors for Azure Logic Apps](built-in.md)
- [Microsoft Graph Security API overview](/graph/security-concept-overview)
- [What is Azure Logic Apps?](../logic-apps/logic-apps-overview.md)
- [What is Power Automate?](https://make.powerautomate.com/)
- [What is Power Apps?](https://powerapps.microsoft.com/)
