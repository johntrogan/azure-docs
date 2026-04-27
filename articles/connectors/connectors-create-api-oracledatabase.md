---
title: Connect to Oracle Database from Workflows
description: Access and run tasks in your Oracle database by using automation and integration workflows in Azure Logic Apps.
services: azure-logic-apps
ms.suite: integration
ms.reviewers: estfan, azla
ms.topic: how-to
ai-usage: ai-assisted
ms.update-cycle: 365-days
ms.date: 04/27/2026
# Customer intent: As an automation and integration developer who works with Azure Logic Apps, I want to connect to an Oracle database from my workflow to perform management operations.
---

# Connect to Oracle databases from workflows in Azure Logic Apps

[!INCLUDE [logic-apps-sku-consumption-standard](../../includes/logic-apps-sku-consumption-standard.md)]

When your integration and automation workflows need to work with Oracle data, connect to your Oracle databases by using the **Oracle Database** connector in Azure Logic Apps. You can access databases hosted either on-premises or on an Azure virtual machine.

The **Oracle Database** connector helps you solve common data integration tasks, for example:

- Add customer records to your database.
- Update order records in your database.
- Get, insert, or delete table rows as part of your workflow.

## Supported Oracle Database versions

The following table lists the supported Oracle DB versions that each connector supports:

| Connector | Logic app | Supported Oracle DB versions |
|-----------|-----------|------------------------------|
| Managed | - Consumption <br>- Standard | - Oracle 9 and later <br>- Oracle Data Access Client (ODAC) 11.2 and later |
| Built-in (preview) | Standard | Oracle Database 11 and later |

## Connector technical reference

The Oracle Database connector has different versions, based on [logic app workflow type and host environment](../logic-apps/logic-apps-overview.md#resource-environment-differences).

| Logic app | Environment | Connector version |
|-----------|-------------|-------------------|
| **Consumption** | Multitenant Azure Logic Apps | Managed connector, which appears in the connector gallery under the **Shared** filter. <br><br>For more information, see [Oracle Database managed connector reference](/connectors/oracle/). |
| **Standard** | Single-tenant Azure Logic Apps, App Service Environment v3 (Windows plans only), and Hybrid | Managed connector, which appears in the connector gallery under the **Shared** filter, and built-in connector (public preview), which appears in the connector gallery under the **Built-in** filter. <br><br>The built-in version runs in-process with the Azure Logic Apps runtime and doesn't require the on-premises data gateway because the runtime can reach your Oracle endpoint over the network. <br><br>For more information, see [Oracle Database managed connector reference](/connectors/oracle/) and [Built-in connectors in Azure Logic Apps](built-in.md). |

### Built-in connector operations (preview)

The built-in connector currently supports the following actions:

- **Get tables**
- **Get rows**
- **Insert row**
- **Execute query**
- **Execute stored procedure**

## Prerequisites

- An Azure account and subscription. [Get a free Azure account](https://azure.microsoft.com/pricing/purchase-options/azure-account?cid=msft_learn).

- The logic app resource and workflow from where to connect to your Oracle database. 

  This connector provides only actions, not triggers. You can use any trigger that you want to start your workflow. To create the logic app resource and workflow, and then add a trigger, see:

  - [Create Consumption logic app workflows in Azure Logic Apps](../logic-apps/quickstart-create-example-consumption-workflow.md).
  - [Create Standard logic app workflows Azure Logic apps](../logic-apps/create-single-tenant-workflows-azure-portal.md)
  - [Add a trigger to your workflow](../logic-apps/add-trigger-action-workflow.md#add-trigger)

### Managed connector prerequisites (Consumption and Standard)

- [Download and install the on-premises data gateway](/data-integration/gateway/service-gateway-install).

  This gateway acts as a bridge and provides a secure data transfer between on-premises data and your app or client. You can use the same gateway installation with multiple services and data sources, which means you might only need to install the gateway once.

- Install your Oracle client on the computer where you installed the on-premises data gateway. Otherwise, an error occurs when you try to create or use the connection.

- [Create an Azure gateway resource for your gateway installation](../logic-apps/logic-apps-gateway-connection.md).

### Built-in connector prerequisites (Standard, preview)

- Make sure that your Standard logic app workflow can reach your Oracle endpoint, including any host, port, DNS resolution, and firewall rules.

- When you create the Oracle database connection, provide the following values:

  - **Server address**
  - **Username**
  - **Password**

  You can specify the server address in Easy Connect format (host/port/service name) or as a TNS descriptor.

- For the **Get row** action used in this example, you need to know the identifier for the table to access.

  If you don't know this information, contact your Oracle Database administrator, or get the output from the following statement: `select * from <table-name>`.

## Known issues and limitations

The current connector versions don't support triggers. Use any trigger that fits your scenario to start your workflow, and then add Oracle actions.

| Connector | Limitations |
|-----------|-------------|
| Managed | - Tables with composite keys <br>- Tables with nested object types <br>- Database functions with nonscalar values |
| Built-in | - No dedicated update or delete actions. For update and delete scenarios, use the **Execute query** or **Execute stored procedure** actions. <br>- Some connection problems might appear only at workflow runtime, rather than at connection creation time. |

## Add an action

The steps to add and use an Oracle action differ based on whether you use the built-in connector or managed connector.

### Add a built-in connector action (Standard, preview)

1. Follow the [generic steps](../logic-apps/add-trigger-action-workflow.md#add-action.md) to add the **Oracle Database** action you want to your workflow.

   This example continues with the **Get rows** action.

1. In the connection information pane, enter the required information, such as the connection name you want, Oracle database server IP address, username, and password.

1. When you finish, select **Create new**.

1. In the action information pane, enter the parameter values required for your selected action.

   For example, if you select the **Get rows** action, enter the table name.

1. Add any other actions necessary to finish your workflow.

1. Save the workflow. On the designer toolbar, select **Save**.

### Add a managed connector action (Consumption and Standard)

1. Follow the [generic steps](../logic-apps/add-trigger-action-workflow.md#add-action.md) to add the **Oracle Database** action you want to your workflow.

   This example continues with the [**Get rows** action](/connectors/oracle/#get-rows).

1. In the connection information pane, enter the required [connection information](/connectors/oracle/#default).

1. For the **Gateway** property, select the Azure subscription and Azure gateway resource to use.

1. After you finish the connection is complete, from the **Table name** list, select a table.

1. For the **Row Id** property, enter the row ID that you want in your table.

   In the following example, job data is returned from a Human Resources database:

   :::image type="content" source="media/connectors-create-api-oracledatabase/table-rowid.png" alt-text="Screenshot shows the Azure portal, workflow designer, and Get row action with the table name and row ID." lightbox="media/connectors-create-api-oracledatabase/table-rowid.png":::

1. Add any other actions necessary to finish your workflow.

1. Save the workflow. On the designer toolbar, select **Save**.

## Troubleshoot Oracle database connection problems

#### **Error**: Cannot reach the Gateway

**Cause**: The on-premises data gateway can't connect to the cloud.

**Mitigation**: Make sure your gateway is running on the on-premises computer where you installed the gateway and has internet connectivity. Avoid installing the gateway on a computer that might be turned off or go to sleep. You can also try restarting the on-premises data gateway service (PBIEgwService).

#### **Error**: The provider being used is deprecated: 'System.Data.OracleClient requires Oracle client software version 8.1.7 or greater.' To install the official provider, see [https://go.microsoft.com/fwlink/p/?LinkID=272376](/power-bi/connect-data/desktop-connect-oracle-database).

**Cause**: The Oracle client SDK isn't installed on the computer where the on-premises data gateway is running.

**Resolution**: Download and install the Oracle client SDK on the same computer as the on-premises data gateway.

#### **Error**: Table '[Tablename]' does not define any key columns

**Cause**: The table doesn't have a primary key.

**Resolution**: The Oracle Database connector requires that you use a table with a primary key column.

## Related content

- [Managed connectors for Azure Logic Apps](managed.md)
- [Built-in connectors for Azure Logic Apps](built-in.md)
