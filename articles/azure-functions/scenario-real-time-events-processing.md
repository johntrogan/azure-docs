---
title: Process real-time events using Azure Functions
description: "Learn how to use the Azure Developer CLI (azd) to create resources and deploy a real-time event processing project to a Flex Consumption plan on Azure."
ms.date: 02/08/2026
ms.topic: quickstart
ai-usage: ai-assisted
zone_pivot_groups: programming-languages-set-functions
#Customer intent: As a developer, I need to know how to use the Azure Developer CLI to create and deploy an Event Hubs triggered function for real-time event processing to a new function app in the Flex Consumption plan in Azure.
---

# Quickstart: Process real-time events by using Azure Functions

In this article, you use the Azure Developer CLI (`azd`) to create an Event Hubs trigger function for real-time event processing in Azure Functions. After verifying the code locally, you deploy it to a new serverless function app running in a Flex Consumption plan in Azure.

The project source uses `azd` to create the function app and related resources and to deploy your code to Azure. This deployment follows current best practices for secure and scalable Azure Functions deployments.

By default, the Flex Consumption plan follows a _pay-for-what-you-use_ billing model, which means you can complete this article and only incur a small cost of a few USD cents or less in your Azure account.
::: zone pivot="programming-language-java,programming-language-javascript,programming-language-powershell"  
> [!IMPORTANT]  
> While [processing Event Hubs messages](./functions-bindings-event-hubs.md) is supported for all languages, this quickstart scenario currently only has examples for C#, Python, and TypeScript. To complete this quickstart, select one of these supported languages at the top of the article. 
::: zone-end  
::: zone pivot="programming-language-csharp,programming-language-python,programming-language-typescript" 
## Prerequisites
::: zone-end  
::: zone pivot="programming-language-csharp"  
+ [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)
::: zone-end  
::: zone pivot="programming-language-typescript"
+ [Node.js 22](https://nodejs.org/) or later  
::: zone-end  
::: zone pivot="programming-language-python" 
+ [Python 3.11](https://www.python.org/) or later
::: zone-end  
::: zone pivot="programming-language-csharp,programming-language-python,programming-language-typescript" 
+ [Azurite storage emulator](../storage/common/storage-use-azurite.md)

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
    azd init --template functions-quickstart-dotnet-azd-eventhub -e eventhub-dotnet
    ```

    This command pulls the project files from the [template repository](https://github.com/Azure-Samples/functions-quickstart-dotnet-azd-eventhub) and initializes the project in the current folder. The `-e` flag sets a name for the current environment. In `azd`, the environment maintains a unique deployment context for your app, and you can define more than one. The environment name is also used in the name of the resource group you create in Azure. 
::: zone-end  
::: zone pivot="programming-language-typescript"  
1. In your local terminal or command prompt, run this `azd init` command in an empty folder:
 
    ```console
    azd init --template functions-quickstart-typescript-azd-eventhub -e eventhub-ts
    ```

    This command pulls the project files from the [template repository](https://github.com/Azure-Samples/functions-quickstart-typescript-azd-eventhub) and initializes the project in the current folder. The `-e` flag sets a name for the current environment. In `azd`, the environment maintains a unique deployment context for your app, and you can define more than one. The environment name is also used in the name of the resource group you create in Azure.
::: zone-end  
::: zone pivot="programming-language-python"  
1. In your local terminal or command prompt, run this `azd init` command in an empty folder:
 
    ```console
    azd init --template functions-quickstart-python-azd-eventhub -e eventhub-py
    ```
        
    This command pulls the project files from the [template repository](https://github.com/Azure-Samples/functions-quickstart-python-azd-eventhub) and initializes the project in the current folder. The `-e` flag sets a name for the current environment. In `azd`, the environment maintains a unique deployment context for your app, and you can define more than one. The environment name is also used in the name of the resource group you create in Azure.

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

::: zone-end
::: zone pivot="programming-language-csharp,programming-language-python,programming-language-typescript" 
## Provision Azure resources

Before you can run your function locally, you need to provision an Event Hubs namespace and hub in Azure. Use `azd provision` to create these resources and configure your local settings by adding the required *local.settings.json* file.

1. Run this command to sign in to Azure:

    ```console
    azd auth login
    ```

    Follow the prompts to authenticate with your Azure account.

1. From the root folder, run this command to provision the Azure resources:

    ```console
    azd provision
    ```

1. When prompted, provide these required deployment parameters:

    | Parameter | Description |
    | ---- | ---- |
    | _Azure subscription_ | Subscription in which you create your resources.|
    | _Azure location_ | Azure region in which to create the resource group that contains the new Azure resources. Only regions that currently support the Flex Consumption plan are shown. |

    The `azd provision` command creates the required Azure resources, including an Event Hubs namespace and hub, a Flex Consumption function app, Application Insights, and a storage account. It also configures your *local.settings.json* file with the Event Hubs connection information.

## Run in your local environment  
::: zone-end  
::: zone pivot="programming-language-csharp,programming-language-python"
1. Run this command from the `src` folder in a terminal or command prompt:

    ```console
    func start
    ```

::: zone-end  
::: zone pivot="programming-language-typescript"  
1. Run these commands from the root folder in a terminal or command prompt:
 
    ```console
    npm install
    npm start  
    ```
::: zone-end  
::: zone pivot="programming-language-csharp,programming-language-python,programming-language-typescript"  
2. When the Functions host starts in your local project folder, it writes information about your functions to the terminal output. 

    This sample includes a Timer trigger function that automatically generates news articles every 10 seconds and sends them to Event Hubs. The Event Hubs trigger function then processes these events and performs sentiment analysis and engagement tracking.

    You see output similar to this:

    <pre>
    [2024-11-10T10:30:15.123Z] Successfully generated and sent 5 news articles to Event Hub
    [2024-11-10T10:30:15.145Z] ✅ Successfully processed article NEWS-20241110-A1B2C3D4 - 'Breaking: Major Discovery in Renewable Energy Technology' by Sarah Johnson
    [2024-11-10T10:30:15.147Z] 🔥 Viral article: NEWS-20241110-E5F6G7H8 - 8,547 views
    [2024-11-10T10:30:15.149Z] 📊 NEWS BATCH SUMMARY: 5 articles | Total Views: 18,432 | Avg Views: 3,686 | Avg Sentiment: 0.34
    </pre>

3. When you're done, press Ctrl+C in the terminal window to stop the `func.exe` host process.
::: zone-end  
::: zone pivot="programming-language-python"  
4. Run `deactivate` to shut down the virtual environment.
::: zone-end  
::: zone pivot="programming-language-csharp,programming-language-python,programming-language-typescript"  
## Review the code (optional)

You can review the code that defines the Event Hubs trigger function:
::: zone-end      
::: zone pivot="programming-language-csharp"  
:::code language="csharp" source="~/functions-event-hub-azd-dotnet/src/EventHubsTrigger.cs" range="1-51" :::

You can review the complete template project [here](https://github.com/Azure-Samples/functions-quickstart-dotnet-azd-eventhub).
::: zone-end  
::: zone pivot="programming-language-typescript" 
:::code language="typescript" source="~/functions-event-hub-azd-typescript/src/functions/EventHubsTrigger.ts" range="1-43" :::

You can review the complete template project [here](https://github.com/Azure-Samples/functions-quickstart-typescript-azd-eventhub).
::: zone-end  
::: zone pivot="programming-language-python" 
:::code language="python" source="~/functions-event-hub-azd-python/function_app.py" range="1-12,85-130" :::

You can review the complete template project [here](https://github.com/Azure-Samples/functions-quickstart-python-azd-eventhub).
::: zone-end  
::: zone pivot="programming-language-csharp,programming-language-python,programming-language-typescript" 
After you verify your function locally, it's time to publish it to Azure. 

## Deploy to Azure
 
This project is configured to use the `azd up` command to deploy your code to a new function app in a Flex Consumption plan in Azure. Since you already provisioned resources, this command deploys your code to the existing function app.

>[!TIP]
>This project includes a set of Bicep files that `azd` uses to create a secure deployment to a Flex Consumption plan that follows best practices.

1. Run the following command to deploy your code project to the function app in Azure:

    ```console
    azd deploy
    ```

    The deployment packages your code and deploys it to the function app. When the command completes, you see links to the resources you created.

## Verify deployment

After deployment completes, your Event Hubs trigger function automatically starts processing events as they arrive in the Event Hub.

1. In the [Azure portal](https://portal.azure.com), go to your new function app.

1. Select **Log stream** from the left menu to monitor your function executions in real-time.

1. You should see log entries that show your Event Hubs trigger function processing events generated by the Timer trigger.

## Redeploy your code

Run the `azd up` command as many times as you need to both provision your Azure resources and deploy code updates to your function app. 

>[!NOTE]
>Deployed code files are always overwritten by the latest deployment package.

Your initial responses to `azd` prompts and any environment variables generated by `azd` are stored locally in your named environment. Use the `azd env get-values` command to review all of the variables in your environment that were used when creating Azure resources. 

## Clean up resources

When you're done working with your function app and related resources, use this command to delete the function app and its related resources from Azure and avoid incurring any further costs:

```console
azd down --no-prompt
```

>[!NOTE]  
>The `--no-prompt` option instructs `azd` to delete your resource group without a confirmation from you. 
>
>This command doesn't affect your local code project. 
::: zone-end

## Related articles

+ [Event Hubs trigger for Azure Functions](functions-bindings-event-hubs-trigger.md)
+ [Azure Functions scenarios](functions-scenarios.md)
+ [Flex Consumption plan](flex-consumption-plan.md)
+ [Azure Developer CLI (azd)](/azure/developer/azure-developer-cli/)
+ [azd reference](/azure/developer/azure-developer-cli/reference)
+ [Azure Functions Core Tools reference](functions-core-tools-reference.md)
+ [Code and test Azure Functions locally](functions-develop-local.md)
