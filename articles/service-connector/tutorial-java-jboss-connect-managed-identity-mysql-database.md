---
title: Access MySQL data from Java JBoss EAP on App Service
description: Connect to Azure Database for MySQL using managed identity from a sample Java JBoss EAP app on Azure App Service.
ms.devlang: java
ms.topic: tutorial
ms.date: 04/16/2026
ms.service: service-connector
author: xfz11
ms.author: xiaofanzhou
ms.custom:
  - passwordless-java
  - service-connector
  - devx-track-azurecli
  - devx-track-extended-java
  - sfi-ga-nochange
---

# Tutorial: Connect to a MySQL database from Java JBoss EAP on Azure App Service

In this tutorial, you learn how to connect a Java JBoss EAP app on [Azure App Service](/azure/app-service/overview) to an Azure Database for MySQL database using a managed identity. App Service is a scalable, self-patching Azure web hosting service that can use a [managed identity](/azure/app-service/overview-managed-identity) to provide secure access to [Azure Database for MySQL](/azure/mysql/) and other Azure services. A managed identity eliminates the need to use secrets in your app, such as credentials in the environment variables.

This tutorial uses Azure CLI commands to complete the following tasks:

> [!div class="checklist"]
> * Creates an Azure Database for MySQL server and database.
> * Deploys a sample JBoss EAP app to App Service using a WAR package.
> * Configures a Spring Boot web application to use Microsoft Entra authentication with the MySQL database.
> * Connects the web app to the MySQL database using Service Connector with managed identity authentication.

## Prerequisites

- An Azure subscription with write and role-assignment permissions for the tutorial resources, in an Azure region that [supports Service Connector](concept-region-support.md) and has sufficient [App Service support and quota](/azure/azure-resource-manager/management/azure-subscription-service-limits#azure-app-service-limits).

- The `Microsoft.ServiceLinker` resource provider registered in your subscription. If not, run `az provider register -n Microsoft.ServiceLinker` to register the provider.

- [Azure Cloud Shell](/azure/cloud-shell/overview) to run the tutorial steps, or if you prefer to run locally:
  1. Install [Azure CLI](/cli/azure/install-azure-cli) 2.30.0 or higher. To check your version, run `az --version`. To upgrade, run `az upgrade`.
  1. Sign in to Azure by using `az login` and following the prompts. If you have more than one subscription connected to your login credentials, run `az account set --subscription <subscription-ID>` to select a subsciption.


* [Git](https://git-scm.com/)
* [Java JDK](/azure/developer/java/fundamentals/java-support-on-azure)
* [Maven](https://maven.apache.org)
* [Azure CLI](/cli/azure/install-azure-cli) version 2.46.0 or higher.
* [Azure CLI serviceconnector-passwordless extension](/cli/azure/azure-cli-extensions-list) version 0.2.2 or higher.
* [jq](https://jqlang.github.io/jq/)

## Set up your environment

1. Install the following Azure CLI extensions:

   ```azurecli
   az extension add --name serviceconnector-passwordless --upgrade
   az extension add --name rdbms-connect
   ```

1. Run the following commands to clone the sample repo and change directories into the sample app project. Run all remaining commands from this folder.

   ```bash
   git clone https://github.com/Azure-Samples/Passwordless-Connections-for-Java-Apps
   cd Passwordless-Connections-for-Java-Apps/JakartaEE/jboss-eap/
   ```

1. Define the following environment variables for the tutorial, replacing the `<region>` placeholder with a valid value. `LOCATION` must be an Azure region where your subscription has sufficient quota to create the Azure resources and no restrictions on any of the services.

   ```bash
   LOCATION="<region>"
   RESOURCE_GROUP_NAME="mysql-mi-webapp"
   ```

1. Create a [resource group](/azure/azure-resource-manager/management/overview#terminology) to contain all the project resources. The resource group name is cached and automatically applied to subsequent commands.

    ```azurecli
    az group create --name $RESOURCE_GROUP_NAME --location $LOCATION
    ```

## Create an Azure Database for MySQL

Create an Azure Database for MySQL server and database in your subscription. The Spring Boot app connects to this database and stores its data when running, persisting the application state no matter where you run the application.

1. Create an Azure Database for MySQL server. Although the command defines an administrator account, the account isn't used for this tutorial because the Microsoft Entra admin account does all administrative tasks. The `MYSQL_HOST` name must be unique across all of Azure.

   ```azurecli
   export MYSQL_ADMIN_USER=azureuser
   export MYSQL_ADMIN_PASSWORD="AdminPassword1"
   export RAND_ID=$RANDOM
   export MYSQL_HOST="mysql-mi-$RAND_ID"
   az mysql flexible-server create \
       --name $MYSQL_HOST \
       --resource-group $RESOURCE_GROUP \
       --location $LOCATION \
       --admin-user $MYSQL_ADMIN_USER \
       --admin-password $MYSQL_ADMIN_PASSWORD \
       --public-access 0.0.0.0 \
       --tier Burstable \
       --sku-name Standard_B1ms \
       --storage-size 32
   ```

1. Create a database for the application to use.

   ```azurecli
   export DATABASE_NAME="checklist"
   az mysql flexible-server db create \
       --resource-group $RESOURCE_GROUP \
       --server-name $MYSQL_HOST \
       --database-name $DATABASE_NAME
   ```

## Create an App Service resource

Create an App Service JBoss EAP resource on Linux. JBoss EAP requires a Premium SKU.

```azurecli

# Create an App Service plan
export APPSERVICE_PLAN="mysql-mi-plan"
export APPSERVICE_NAME="mysql-mi-app"
az appservice plan create \
    --resource-group $RESOURCE_GROUP \
    --name $APPSERVICE_PLAN \
    --location $LOCATION \
    --sku P1V3 \
    --is-linux

# Create an App Service web app
az webapp create \
    --resource-group $RESOURCE_GROUP \
    --name $APPSERVICE_NAME \
    --plan $APPSERVICE_PLAN \
    --runtime "JBOSSEAP:7-java8"
```

## Create a user-assigned managed identity and grant it permissions

1. Create a user-assigned managed identity for Microsoft Entra authentication using the following command. For more information, see [Set up Microsoft Entra authentication for Azure Database for MySQL - Flexible Server](/azure/mysql/flexible-server/how-to-azure-ad).

    ```azurecli
    export USER_IDENTITY_NAME=<your-user-assigned-managed-identity-name>
    export IDENTITY_RESOURCE_ID=$(az identity create \
        --name $USER_IDENTITY_NAME \
        --resource-group $RESOURCE_GROUP \
        --query id \
        --output tsv)
    ```

1. Ask your *Global Administrator* or *Privileged Role Administrator* to grant the following permissions to the new user-assigned identity: `User.Read.All`, `GroupMember.Read.All`, and `Application.Read.ALL`. For more information, see the [Permissions](/azure/mysql/flexible-server/concepts-azure-ad-authentication#permissions) section of [Active Directory authentication](/azure/mysql/flexible-server/concepts-azure-ad-authentication).

## Connect the MySQL database using managed identity

Use [Service Connector](overview.md) to connect your app to the MySQL database with a system-assigned managed identity. Service Connector does the following tasks in the background:

* Sets the database Microsoft Entra admin to the current signed-in user.
* Enables system-assigned managed identity for the app `$APPSERVICE_NAME` hosted by Azure App Service.
* Adds a database user for the system-assigned managed identity and grants all privileges of the database `$DATABASE_NAME` to this user. You can get the user name from the connection string in the output from the previous command.
* Adds a connection string to App Settings in the app named `AZURE_MYSQL_CONNECTIONSTRING`.

Use the [az webapp connection create](/cli/azure/webapp/connection/create#az-webapp-connection-create-mysql-flexible) command to connect your app to the MySQL database with a system-assigned managed identity.

```azurecli
az webapp connection create mysql-flexible \
    --resource-group $RESOURCE_GROUP \
    --name $APPSERVICE_NAME \
    --target-resource-group $RESOURCE_GROUP \
    --server $MYSQL_HOST \
    --database $DATABASE_NAME \
    --system-identity mysql-identity-id=$IDENTITY_RESOURCE_ID \
    --client-type java
```

## Prepare data in the database

1. Open a firewall to allow connection from your current IP address.

   ```azurecli
   # Create a temporary firewall rule to allow connections from your current machine to the MySQL server
   export MY_IP=$(curl http://whatismyip.akamai.com)
   az mysql flexible-server firewall-rule create \
       --resource-group $RESOURCE_GROUP \
       --name $MYSQL_HOST \
       --rule-name AllowCurrentMachineToConnect \
       --start-ip-address ${MY_IP} \
       --end-ip-address ${MY_IP}
   ```

1. Connect to the database and create tables.

   ```azurecli
   export DATABASE_FQDN=${MYSQL_HOST}.mysql.database.azure.com
   export CURRENT_USER=$(az account show --query user.name --output tsv)
   export RDBMS_ACCESS_TOKEN=$(az account get-access-token \
       --resource-type oss-rdbms \
       --output tsv \
       --query accessToken)
   mysql -h "${DATABASE_FQDN}" --user "${CURRENT_USER}" --enable-cleartext-plugin --password="$RDBMS_ACCESS_TOKEN" < azure/init-db.sql
   ```

1. Remove the temporary firewall rule.

   ```azurecli
   az mysql flexible-server firewall-rule delete \
       --resource-group $RESOURCE_GROUP \
       --name $MYSQL_HOST \
       --rule-name AllowCurrentMachineToConnect
   ```

## Deploy the application

1. Update the connection string in App Settings.

   Get the connection string generated by Service Connector and add passwordless authentication plugin. This connection string is referenced in the startup script.

   ```azurecli
   export PASSWORDLESS_URL=$(\
       az webapp config appsettings list \
           --resource-group $RESOURCE_GROUP \
           --name $APPSERVICE_NAME \
       | jq -c '.[] \
       | select ( .name == "AZURE_MYSQL_CONNECTIONSTRING" ) \
       | .value' \
       | sed 's/"//g')
   # Create a new environment variable with the connection string including the passwordless authentication plugin
   export PASSWORDLESS_URL=${PASSWORDLESS_URL}'&defaultAuthenticationPlugin=com.azure.identity.extensions.jdbc.mysql.AzureMysqlAuthenticationPlugin&authenticationPlugins=com.azure.identity.extensions.jdbc.mysql.AzureMysqlAuthenticationPlugin'
   az webapp config appsettings set \
       --resource-group $RESOURCE_GROUP \
       --name $APPSERVICE_NAME \
       --settings "AZURE_MYSQL_CONNECTIONSTRING_PASSWORDLESS=${PASSWORDLESS_URL}"
   ```

1. The sample app contains a *pom.xml* file that can generate the WAR file. Run the following command to build the app.

   ```bash
   mvn clean package -DskipTests
   ```

1. Deploy the WAR and the startup script to App Service.

   ```azurecli
   az webapp deploy \
       --resource-group $RESOURCE_GROUP \
       --name $APPSERVICE_NAME \
       --src-path target/ROOT.war \
       --type war
   az webapp deploy \
       --resource-group $RESOURCE_GROUP \
       --name $APPSERVICE_NAME \
       --src-path src/main/webapp/WEB-INF/createMySQLDataSource.sh \
       --type startup
   ```

## Test the sample web app

Run the following code to test the application:

```bash
export WEBAPP_URL=$(az webapp show \
    --resource-group $RESOURCE_GROUP \
    --name $APPSERVICE_NAME \
    --query defaultHostName \
    --output tsv)

# Create a list
curl -X POST -H "Content-Type: application/json" -d '{"name": "list1","date": "2022-03-21T00:00:00","description": "Sample checklist"}' https://${WEBAPP_URL}/checklist

# Create few items on the list 1
curl -X POST -H "Content-Type: application/json" -d '{"description": "item 1"}' https://${WEBAPP_URL}/checklist/1/item
curl -X POST -H "Content-Type: application/json" -d '{"description": "item 2"}' https://${WEBAPP_URL}/checklist/1/item
curl -X POST -H "Content-Type: application/json" -d '{"description": "item 3"}' https://${WEBAPP_URL}/checklist/1/item

# Get all lists
curl https://${WEBAPP_URL}/checklist

# Get list 1
curl https://${WEBAPP_URL}/checklist/1
```

[!INCLUDE [cli-samples-clean-up](../../includes/cli-samples-clean-up.md)]

## Next step

Learn more about running Java apps on App Service on Linux in the developer guide.

> [!div class="nextstepaction"]
> [Java in App Service Linux dev guide](../app-service/configure-language-java-security.md?pivots=platform-linux)
