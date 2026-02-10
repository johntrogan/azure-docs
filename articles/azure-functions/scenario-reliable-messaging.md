---
title: Reliable messaging with Azure Functions and Service Bus
description: "Learn how to use the Azure Developer CLI (azd) to create resources and deploy a Service Bus trigger project to a Flex Consumption plan on Azure."
ms.date: 02/09/2026
ms.topic: quickstart
ai-usage: ai-assisted
zone_pivot_groups: programming-languages-set-functions
#Customer intent: As a developer, I need to know how to use the Azure Developer CLI to create and deploy reliable messaging solutions using Service Bus triggers to a new function app in the Flex Consumption plan in Azure.
---

# Quickstart: Reliable messaging with Azure Functions and Service Bus

In this article, you use the Azure Developer CLI (`azd`) to create a Service Bus trigger function for reliable queue-based messaging in Azure Functions. After verifying the code locally, you deploy it to a new serverless function app you create running in a Flex Consumption plan in Azure Functions.

The project source uses `azd` to create the function app and related resources and to deploy your code to Azure. This deployment follows current best practices for secure and scalable Azure Functions deployments.

By default, the Flex Consumption plan follows a _pay-for-what-you-use_ billing model, which means you can complete this article and only incur a small cost of a few USD cents or less in your Azure account.
::: zone pivot="programming-language-javascript,programming-language-powershell"  
> [!IMPORTANT]  
> While [Service Bus triggers](./functions-bindings-service-bus-trigger.md) are supported for all languages, this quickstart scenario currently only has examples for C#, Python, TypeScript, and Java. To complete this quickstart, select one of these supported languages at the top of the article. 
::: zone-end  
::: zone pivot="programming-language-csharp,programming-language-python,programming-language-typescript,programming-language-java" 
## Prerequisites
::: zone-end  
::: zone pivot="programming-language-csharp"  
+ [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)
::: zone-end  
::: zone pivot="programming-language-java"  
+ [Java 17 Developer Kit](/azure/developer/java/fundamentals/java-support-on-azure)
        + If you use another [supported version of Java](supported-languages.md?pivots=programming-language-java#languages-by-runtime-version), update the project configuration.  
    + Set the `JAVA_HOME` environment variable to the install location of the correct version of the Java Development Kit (JDK).
+ [Maven 3.6 or later](https://maven.apache.org/download.cgi)
::: zone-end  
::: zone pivot="programming-language-typescript"
+ [Node.js 18](https://nodejs.org/) or above  
::: zone-end  
::: zone pivot="programming-language-python" 
+ [Python 3.8](https://www.python.org/) or above
::: zone-end  
::: zone pivot="programming-language-csharp,programming-language-python,programming-language-typescript,programming-language-java" 
+ [Azure Developer CLI](/azure/developer/azure-developer-cli/install-azd)

+ [Azure Functions Core Tools](functions-run-local.md#install-the-azure-functions-core-tools)

+ [Azure CLI](/cli/azure/install-azure-cli)

+ An Azure account with an active subscription. [Create an account for free](https://azure.microsoft.com/pricing/purchase-options/azure-account?cid=msft_learn).

## Initialize the project

Use the `azd init` command to create a local Azure Functions code project from a template.
::: zone-end 
::: zone pivot="programming-language-csharp"  
1. In your local terminal or command prompt, run this `azd init` command in an empty folder:
 
    ```console
    azd init --template functions-quickstart-dotnet-azd-service-bus -e servicebus-dotnet
    ```

    This command pulls the project files from the [template repository](https://github.com/Azure-Samples/functions-quickstart-dotnet-azd-service-bus) and initializes the project in the current folder. The `-e` flag sets a name for the current environment. In `azd`, the environment maintains a unique deployment context for your app, and you can define more than one. The environment name is also used in the name of the resource group you create in Azure. 

1. Run this command to go to the app folder:

    ```console
    cd src
    ```

1. Create a file named _local.settings.json_ in the `src` folder that contains this JSON data:

    ```json
    {
        "IsEncrypted": false,
        "Values": {
            "AzureWebJobsStorage": "UseDevelopmentStorage=true",
            "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
            "ServiceBusConnection": "",
            "ServiceBusQueueName": "testqueue"
        }
    }
    ```

    This file is required when running locally. The `ServiceBusConnection` is empty for local development. You need an actual Service Bus connection for full testing, which the deployment to Azure provides.

1. Restore the .NET packages:

    ```console
    dotnet restore
    ```
::: zone-end  
::: zone pivot="programming-language-typescript"  
1. In your local terminal or command prompt, run this `azd init` command in an empty folder:
 
    ```console
    azd init --template functions-quickstart-typescript-azd-service-bus -e servicebus-ts
    ```

    This command pulls the project files from the [template repository](https://github.com/Azure-Samples/functions-quickstart-typescript-azd-service-bus) and initializes the project in the current folder. The `-e` flag sets a name for the current environment. In `azd`, the environment maintains a unique deployment context for your app, and you can define more than one. The environment name is also used in the name of the resource group you create in Azure. 

1. Run this command to go to the app folder:

    ```console
    cd src
    ```

1. Create a file named _local.settings.json_ in the `src` folder that contains this JSON data:

    ```json
    {
        "IsEncrypted": false,
        "Values": {
            "AzureWebJobsStorage": "UseDevelopmentStorage=true",
            "FUNCTIONS_WORKER_RUNTIME": "node",
            "ServiceBusConnection": "",
            "ServiceBusQueueName": "testqueue"
        }
    }
    ```

    This file is required when running locally. The `ServiceBusConnection` is empty for local development. You need an actual Service Bus connection for full testing, which the deployment to Azure provides.

1. Install the required Node.js packages and build the TypeScript code:

    ```console
    npm install
    npm run build
    ```
::: zone-end  
::: zone pivot="programming-language-python"  
1. In your local terminal or command prompt, run this `azd init` command in an empty folder:
 
    ```console
    azd init --template functions-quickstart-python-azd-service-bus -e servicebus-py
    ```
        
    This command pulls the project files from the [template repository](https://github.com/Azure-Samples/functions-quickstart-python-azd-service-bus) and initializes the project in the current folder. The `-e` flag sets a name for the current environment. In `azd`, the environment maintains a unique deployment context for your app, and you can define more than one. The environment name is also used in the name of the resource group you create in Azure. 

1. Run this command to go to the app folder:

    ```console
    cd src
    ```

1. Create a file named _local.settings.json_ in the `src` folder that contains this JSON data:

    ```json
    {
        "IsEncrypted": false,
        "Values": {
            "AzureWebJobsStorage": "UseDevelopmentStorage=true",
            "FUNCTIONS_WORKER_RUNTIME": "python",
            "ServiceBusConnection": "",
            "ServiceBusQueueName": "testqueue"
        }
    }
    ```

    This file is required when running locally. The `ServiceBusConnection` is empty for local development. You need an actual Service Bus connection for full testing, which the deployment to Azure provides.

## Create and activate a virtual environment

In the root folder, run these commands to create and activate a virtual environment named `.venv`:

### [Linux/macOS](#tab/linux)

```bash
python3 -m venv .venv
source .venv/bin/activate
```

If Python doesn't install the venv package on your Linux distribution, run the following command:

```bash
sudo apt-get install python3-venv
```

### [Windows (bash)](#tab/windows-bash)

```bash
py -m venv .venv
source .venv/scripts/activate
```

### [Windows (Cmd)](#tab/windows-cmd)

```shell
py -m venv .venv
.venv\scripts\activate
```

---

After activating the virtual environment, install the required Python packages:

```shell
pip install -r src/requirements.txt
```
::: zone-end  
::: zone pivot="programming-language-java"  
1. In your local terminal or command prompt, run this `azd init` command in an empty folder:
 
    ```console
    azd init --template functions-quickstart-java-azd-service-bus -e servicebus-java
    ```
        
    This command pulls the project files from the [template repository](https://github.com/Azure-Samples/functions-quickstart-java-azd-service-bus) and initializes the project in the current folder. The `-e` flag sets a name for the current environment. In `azd`, the environment maintains a unique deployment context for your app, and you can define more than one. The environment name is also used in the name of the resource group you create in Azure. 

1. Run this command to go to the app folder:

    ```console
    cd src
    ```

1. Create a file named _local.settings.json_ in the `src` folder that contains this JSON data:

    ```json
    {
        "IsEncrypted": false,
        "Values": {
            "AzureWebJobsStorage": "UseDevelopmentStorage=true",
            "FUNCTIONS_WORKER_RUNTIME": "java",
            "ServiceBusConnection": "",
            "ServiceBusQueueName": "testqueue"
        }
    }
    ```

    This file is required when running locally. The `ServiceBusConnection` is empty for local development. You need an actual Service Bus connection for full testing, which the deployment to Azure provides.
::: zone-end

::: zone pivot="programming-language-csharp,programming-language-python,programming-language-typescript,programming-language-java" 
## Run in your local environment  
::: zone-end  
::: zone pivot="programming-language-csharp,programming-language-python" 
1. Run this command from your app folder in a terminal or command prompt:

    ```console
    func start
    ``` 
::: zone-end  
::: zone pivot="programming-language-java" 
1. Run these commands from your app folder in a terminal or command prompt:
 
    ```console
    mvn clean package
    mvn azure-functions:run
    ```
::: zone-end  
::: zone pivot="programming-language-typescript"  
1. Run this command from your app folder in a terminal or command prompt:
 
    ```console
    npm start  
    ```
::: zone-end  
::: zone pivot="programming-language-csharp,programming-language-python,programming-language-typescript,programming-language-java" 
2. When the Functions host starts in your local project folder, it writes information about your Service Bus triggered function to the terminal output.

    > [!NOTE]
    > Since this function uses a Service Bus trigger, it starts but doesn't process messages until it connects to an actual Service Bus queue. The function is ready and waiting for messages.

3. When you're done, press Ctrl+C in the terminal window to stop the `func.exe` host process.
::: zone-end 
::: zone pivot="programming-language-python"
4. Run `deactivate` to shut down the virtual environment.
::: zone-end
::: zone pivot="programming-language-csharp,programming-language-python,programming-language-typescript,programming-language-java" 
## Review the code (optional)

You can review the code that defines the Service Bus trigger function:
::: zone-end      
::: zone pivot="programming-language-csharp"  
:::code language="csharp" source="~/functions-quickstart-dotnet-azd-service-bus/src/ServiceBusQueueTrigger.cs" range="1-27" :::

You can review the complete template project [here](https://github.com/Azure-Samples/functions-quickstart-dotnet-azd-service-bus).
::: zone-end  
::: zone pivot="programming-language-typescript" 
:::code language="typescript" source="~/functions-quickstart-typescript-azd-service-bus/src/index.ts" range="1-17" :::

You can review the complete template project [here](https://github.com/Azure-Samples/functions-quickstart-typescript-azd-service-bus).
::: zone-end  
::: zone pivot="programming-language-python" 
:::code language="python" source="~/functions-quickstart-python-azd-service-bus/src/function_app.py" range="1-13" :::

You can review the complete template project [here](https://github.com/Azure-Samples/functions-quickstart-python-azd-service-bus).
::: zone-end  
::: zone pivot="programming-language-java" 
:::code language="java" source="~/functions-quickstart-java-azd-service-bus/src/main/java/com/example/ServiceBusFunction.java" range="1-36" :::

You can review the complete template project [here](https://github.com/Azure-Samples/functions-quickstart-java-azd-service-bus).
::: zone-end  
::: zone pivot="programming-language-csharp,programming-language-python,programming-language-typescript,programming-language-java" 
After you verify your function locally, it's time to publish it to Azure. 

## Deploy to Azure
 
This project is configured to use the `azd up` command to deploy your code to a new function app in a Flex Consumption plan in Azure.

>[!TIP]
>This project includes a set of Bicep files that `azd` uses to create a secure deployment to a Flex consumption plan that follows best practices.

1. Run this command to have `azd` create the required Azure resources in Azure and deploy your code project to the new function app:

    ```console
    azd up
    ```

    The root folder contains the `azure.yaml` definition file required by `azd`. 

    If you're not already signed in, you're asked to authenticate by using your Azure account.

1. When prompted, provide these required deployment parameters:

    | Parameter | Description |
    | ---- | ---- |
    | _Azure subscription_ | Subscription in which you create your resources.|
    | _Azure location_ | Azure region in which to create the resource group that contains the new Azure resources. Only regions that currently support the Flex Consumption plan are shown.|
    
    The `azd up` command uses your response to these prompts with the Bicep configuration files to complete these deployment tasks:

    + Create and configure these required Azure resources (equivalent to `azd provision`):

        + Flex Consumption plan and function app
        + Azure Storage (required) and Application Insights (recommended)
        + Service Bus namespace and queue with private endpoint
        + Access policies and roles for your account
        + Service-to-service connections by using managed identities (instead of stored connection strings)
        + Virtual network to securely run both the function app and the other Azure resources

    + Package and deploy your code to the deployment container (equivalent to `azd deploy`). The app is then started and runs in the deployed package. 

    After the command completes successfully, you see links to the resources you created. 

## Verify deployment

After deployment finishes, your Service Bus trigger function is ready to process messages from the queue.

1. Configure your client IP address in the Service Bus firewall to send test messages. In the Azure portal, go to your Service Bus namespace and select **Networking** > **Public access**. Add your client IP address to the firewall.

1. Use the Service Bus Explorer in the Azure portal to send messages to the Service Bus queue. Follow [Use Service Bus Explorer to run data operations on Service Bus](/azure/service-bus-messaging/explorer) to send messages and peek messages from the queue.

1. Open Application Insights live metrics to monitor the function executions. Send messages (for example, 1,000 messages) and observe the number of instances ('servers online'). Notice your app scaling the number of instances to handle processing the messages.

The sample telemetry shows that your messages are triggering the function and making their way from Service Bus through the virtual network into the function app for processing.

## Redeploy your code

Run the `azd up` command as many times as you need to both provision your Azure resources and deploy code updates to your function app. 

>[!NOTE]
>Deployed code files are always overwritten by the latest deployment package.

Your initial responses to `azd` prompts and any environment variables generated by `azd` are stored locally in your named environment. Use the `azd env get-values` command to review all of the variables in your environment that were used when creating Azure resources. 

## Clean up resources

When you finish working with your function app and related resources, use this command to delete the function app and its related resources from Azure. This step helps you avoid incurring any further costs:

```console
azd down --no-prompt
```

>[!NOTE]  
> The `--no-prompt` option instructs `azd` to delete your resource group without a confirmation from you. 
>
> This command doesn't affect your local code project. 
::: zone-end

## Related articles

+ [Service Bus trigger for Azure Functions](functions-bindings-service-bus-trigger.md)
+ [Azure Functions scenarios](functions-scenarios.md)
+ [Flex Consumption plan](flex-consumption-plan.md)
+ [Azure Developer CLI (azd)](/azure/developer/azure-developer-cli/)
+ [azd reference](/azure/developer/azure-developer-cli/reference)
+ [Azure Functions Core Tools reference](functions-core-tools-reference.md)
+ [Code and test Azure Functions locally](functions-develop-local.md)
