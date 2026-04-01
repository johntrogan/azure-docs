---
title: Create service connections using IaC tools
description: Learn how to create service connections and translate your infrastructure configurations into Infrastructure as Code (IaC) templates to use in CI/CD pipelines.
author: houk-ms
ms.service: service-connector
ms.topic: how-to
ms.date: 04/01/2026
ms.author: honc
#customer intent: As an Azure developer, I want to learn how to create service connections and infrastructure configurations as Infrastructure as Code (IaC) templates so I can use the templates for CI/CD pipelines.
---
# Create service connections using IaC tools

Service Connector helps you quickly and easily connect your compute services to target backing services. When you move from getting-started to a production stage, you also need to transition from manual configurations to Infrastructure as Code (IaC) templates for your continuous integration/continuous delivery (CI/CD) pipelines. This article shows how you can translate your connected Azure services to IaC templates.

## Solution options

Translating Service Connector infrastructure to IaC templates involves implementing the logic to provision source and target services and the logic to build the connections. The Bicep templates in this article create a web app and a storage account and connect them via a system-assigned identity, either in Service Connector or by using template logic. To use these templates, you should understand IaC tools, template authoring grammar, and [known Service Connector IaC limitations](known-limitations.md).

To implement logic to provision source and target services, you can author a template from scratch, or export a template from Azure and polish it. To build a service connection in the template, you can use Service Connector with or without App Configuration, or use template logic directly.

Combinations of these different options produce different solutions. The following table presents the solutions from most to least recommended, based on susceptibility to Service Connector [IaC limitations](known-limitations.md). **Liveness check** refers to whether the solution performs a liveness check on cloud resources before allowing live traffic.

| Solution | Source and target provisioning|Connection creation|Liveness check?| Advantages| Disadvantages |
| ------ | ------------------------ | ----------------------------------------------------- | ---------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
|1| Author from scratch | Service Connector / App Configuration | Yes | - Template simplicity and readability<br />- Service Connector adds value<br />- No Service Connector IaC issues | - Extra dependency to read from App Configuration<br />- Cost of cloud resources liveness check |
|2| Author from scratch |Service Connector | Yes | - Template simplicity and readability<br />- Service Connector adds value | - Cost of cloud resources liveness check<br />- Potential Service Connector IaC issues |
|3| Author from scratch | Directly in template | No | - Template simplicity and readability<br />- No Service Connector IaC issues | - No Service Connector features |
|4| Export and polish | Service Connector / App Configuration | Yes | - Same resources as in the cloud <br />- Service Connector adds value <br />- No Service Connector IaC issues | - Extra dependency to read from App Configuration<br />- Cost of cloud resources liveness check <br />- Effort to understand and polish the template |
|5| Export and polish |Service Connector | Yes | - Same resources as in the cloud <br />- Service Connector adds value | - Cost of cloud resources liveness check <br />- Potential Service Connector IaC issues <br />- Effort to understand and polish the template |
|6| Export and polish | Directly in template | No | - Same resources as in the cloud<br />- No Service Connector IaC issues | - Effort to understand and polish the template <br />- No Service Connector features |

## Provision source and target services

Authoring a template from scratch is the preferred and recommended way to provision source and target services. This method makes it easy to get started and creates a simple, readable template.

If you're provisioning the same resources you have in the cloud, exporting a template from Azure is another option. To export a template, the resource must already exist in Azure.

### Author a template from scratch

The following example template uses a minimal set of parameters to create an Azure web app and storage account.

```bicep
// This template uses a minimal set of parameters to create a web app and a storage account.

param location string = resourceGroup().location
// App Service plan parameters
param planName string = 'plan_${uniqueString(resourceGroup().id)}'
param kind string = 'linux'
param reserved bool = true
param sku string = 'B1'
// Webapp parameters
param webAppName string = 'webapp-${uniqueString(resourceGroup().id)}'
param linuxFxVersion string = 'PYTHON|3.8'
param identityType string = 'SystemAssigned'
param appSettings array = []
// Storage account parameters
param storageAccountName string = 'account${uniqueString(resourceGroup().id)}'

// Create an app service plan 
resource appServicePlan 'Microsoft.Web/serverfarms@2022-09-01' = {
  name: planName
  location: location
  kind: kind
  sku: {
    name: sku
  }
  properties: {
    reserved: reserved
  }
}

// Create a web app
resource appService 'Microsoft.Web/sites@2022-09-01' = {
  name: webAppName
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: linuxFxVersion
      appSettings: appSettings
    }
  }
  identity: {
    type: identityType
  }
}

// Create a storage account
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
}
```

### Export and polish a template

To export a template from an Azure web app, select **Export template** under **Automation** in the app's left navigation menu. The exported template reflects the resource's current states, including Service Connector settings. You can use information from the app's properties to polish the exported template.

:::image type="content" source="./media/how-to/export-webapp-template.png" alt-text="Screenshot of exporting a Bicep template of a web app in the Azure portal.":::

## Create the service connection

You can build the service connection by using Service Connector alone, Service Connector with App Configuration, or directly in template logic.

### Use Service Connector

Creating connections between source and target services using Service Connector is preferred and recommended, as long as Service Connector's [known IaC limitations](known-limitations.md) don't affect your scenario. Using Service Connector simplifies the template and provides features like connection health validation that aren't available when you build connections directly through template logic.

The following example template creates a Service Connector connection between a web app and a storage account, using a system-assigned identity.

```bicep
// This template builds a Service Connector connection between a web app and a storage account using a system-assigned identity.

param webAppName string = 'webapp-${uniqueString(resourceGroup().id)}'
param storageAccountName string = 'account${uniqueString(resourceGroup().id)}'
param connectorName string = 'connector_${uniqueString(resourceGroup().id)}'

// Get an existing webapp
resource webApp 'Microsoft.Web/sites@2022-09-01' existing = {
  name: webAppName
}

// Get an existing storage
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' existing = {
  name: storageAccountName
}

// Create a Service Connector resource for the web app 
// to connect to a storage account using system identity
resource serviceConnector 'Microsoft.ServiceLinker/linkers@2022-05-01' = {
  name: connectorName
  scope: webApp
  properties: {
    clientType: 'python'
    targetService: {
      type: 'AzureResource'
      id: storageAccount.id
    }
    authInfo: {
      authType: 'systemAssignedIdentity'
    }
  }
}
```

For more information about the properties needed to create a Service Connector resource, see [Provide correct parameters to Service Connector](how-to-provide-correct-parameters.md). You can also preview and download an ARM template for reference when you create a Service Connector resource in the Azure portal.

:::image type="content" source="./media/how-to/export-sc-template.png" alt-text="Screenshot of exporting a Bicep template of a service connector resource in the Azure portal.":::

#### Store configuration with App Configuration

App Configuration is the recommended way to store configuration, because it naturally supports IaC scenarios. To add this feature to a Bicep template, add the App Configuration ID in the Service Connector payload. To create an App Configuration connection using the Azure portal, see [Connect Azure services and store configuration in an App Configuration store](tutorial-portal-app-configuration-store.md).

The following example template creates a Service Connector connection between a web app and a storage account, and stores the connection configuration information in App Configuration.

```bicep
resource webApp 'Microsoft.Web/sites@2022-09-01' existing = {
  name: webAppName
}

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' existing = {
  name: storageAccountName
}

resource appConfiguration 'Microsoft.AppConfiguration/configurationStores@2023-03-01' existing = {
  name: appConfigurationName
}

resource serviceConnector 'Microsoft.ServiceLinker/linkers@2022-05-01' = {
  name: connectorName
  scope: webApp
  properties: {
    clientType: 'python'
    targetService: {
      type: 'AzureResource'
      id: storageAccount.id
    }
    authInfo: {
      authType: 'systemAssignedIdentity'
    }
    configurationInfo: {
      configurationStore: {
        appConfigurationId: appConfiguration.id
      }
    }
  }
}
```

### Use template logic

If Service Connector [IaC limitations](./known-limitations.md) affect your scenario, consider building connections directly using template logic. The following template example shows how to connect a storage account to a web app using a system-assigned identity.

```bicep
// This template builds a connection between a web app and a storage account with a system-assigned identity directly

param webAppName string = 'webapp-${uniqueString(resourceGroup().id)}'
param storageAccountName string = 'account${uniqueString(resourceGroup().id)}'
param storageBlobDataContributorRole string  = 'ba92f5b4-2d11-453d-a403-e96b0029c9fe'

// Get an existing webapp
resource webApp 'Microsoft.Web/sites@2022-09-01' existing = {
  name: webAppName
}

// Get an existing storage account
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' existing = {
  name: storageAccountName
}

// Operation: Enable system-assigned identity on the source service
// No action needed as this is enabled when creating the webapp

// Operation: Configure the target service's endpoint on the source service's app settings
resource appSettings 'Microsoft.Web/sites/config@2022-09-01' = {
  name: 'appsettings'
  parent: webApp
  properties: {
    AZURE_STORAGEBLOB_RESOURCEENDPOINT: storageAccount.properties.primaryEndpoints.blob
  }
}

// Operation: Configure firewall on the target service to allow the source service's outbound IPs
// No action needed as storage account allows all IPs by default

// Operation: Create role assignment for the source service's identity on the target service
resource roleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: storageAccount
  name: guid(resourceGroup().id, storageBlobDataContributorRole)
  properties: {
    roleDefinitionId: resourceId('Microsoft.Authorization/roleDefinitions', storageBlobDataContributorRole)
    principalId: webApp.identity.principalId
  }
}
```

#### Authentication operations

Using template logic directly is equivalent to using Service Connector backend operations. When you build connections using template logic, you must implement the same authentication operations that Service Connector would. The following table lists the operations you must translate to template logic for each kind of authentication type.

| Authentication type | Required operations |
| -------------------------------- | ------------------------------------ |
| Secret or connection string       | - Configure the target service connection string on the source service app settings.<br />- Configure firewall on the target service to allow source service outbound IPs. |
| System-assigned managed identity | - Configure the target service endpoint on the source service app settings.<br />- Configure firewall on the target service to allow the source service outbound IPs.<br />- Enable system assigned identity on the source service.<br />- Create role assignment for the source service identity on the target service.|
| User-assigned managed identity   | - Configure the target service endpoint on the source service app settings.<br />- Configure firewall on the target service to allow the source service outbound IPs.<br />- Bind user assigned identity to the source service.<br />- Create role assignment for the user assigned identity on the target service.|
| Service principal | - Configure the target service endpoint on the source service app settings.<br />- Configure the service principal appId and secret on the source service app settings.<br />- Configure firewall on the target service to allow the source service outbound IPs.<br />- Create role assignment for the service principal on the target service. |

