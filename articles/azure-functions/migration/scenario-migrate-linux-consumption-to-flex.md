---
title: "Quickstart: Migrate Linux Consumption apps to Flex Consumption using GitHub Copilot"
description: Use GitHub Copilot with Azure skills to interactively migrate your Linux function apps from the Consumption plan to the Flex Consumption plan.
ms.service: azure-functions
ms.collection:
  - migration
ms.date: 04/07/2026
ms.topic: quickstart

#customer intent: As a developer, I want to use GitHub Copilot to migrate my Linux Consumption plan function apps to the Flex Consumption plan so that I can get better performance and features without manual CLI steps.
---

# Quickstart: Migrate Linux Consumption apps to Flex Consumption using GitHub Copilot

In this quickstart, you use GitHub Copilot with the Azure skills plugin to interactively migrate your Linux function apps from the [Consumption plan](../consumption-plan.md) to the [Flex Consumption plan](../flex-consumption-plan.md). Copilot automates most of the migration, including assessment, app creation, configuration, and deployment.

For a detailed overview of the migration process, supported methods, concepts like dependent services and data protection, and Windows app migration, see [Migrate Consumption plan apps to the Flex Consumption plan](migrate-plan-consumption-to-flex.md).

## Prerequisites

+ An Azure subscription with one or more Linux function apps running on the Consumption plan.

+ The account used for the migration must have the **Owner** or **Contributor** role in the resource group containing your function apps. For the full list of required permissions, see [Prerequisites](migrate-plan-consumption-to-flex.md#prerequisites).

+ [Azure CLI](/cli/azure), version 2.77.0 or later.

+ Configure GitHub Copilot in your preferred mode:

    [!INCLUDE [functions-copilot-setup](~/includes/functions-copilot-setup.md)]

## Identify and migrate your apps

Use this prompt to start an interactive migration that scans your subscription and lets you choose which apps to migrate:

```
migrate my function apps in azure from consumption to flex consumption
```

Copilot identifies your eligible Linux Consumption apps, lets you choose which ones to migrate, and then walks you through assessment, app creation, configuration, and deployment for each app.

> [!NOTE]
> Copilot automatically assesses your app for region, language stack, and version compatibility with the Flex Consumption plan. If your app has compatibility issues, Copilot explains what needs to change before migration can proceed. For details on what's assessed, see [Assess your existing app](migrate-plan-consumption-to-flex.md#assess-your-existing-app).

## Review dependent services

Before finalizing your migration, consider the effect on services connected to your function app. If your app uses triggers other than HTTP, review the [dependent services guidance](migrate-plan-consumption-to-flex.md#consider-dependent-services) and [mitigations by trigger type](migrate-plan-consumption-to-flex.md#mitigations-by-trigger-type) to protect your data during the cutover.

> [!TIP]
> **Simple HTTP-only app?** If your functions only use HTTP triggers and don't connect to other Azure services, you can skip this section. Just remember to update any clients to point to your new app's URL after migration.

## Configure built-in authentication

If your original app used built-in client authentication (Easy Auth), you need to manually recreate it in your new app. Copilot doesn't automate this step. For instructions, see [Configure built-in authentication](migrate-plan-consumption-to-flex.md#configure-built-in-authentication).

## Deploy your app code

After Copilot creates and configures your new Flex Consumption app, deploy your code. Choose the method that fits your workflow:

+ **Continuous deployment**: Update your existing [Azure Pipelines](../functions-how-to-azure-devops.md) or [GitHub Actions](../functions-how-to-github-actions.md) workflows to deploy to the new app. For more information, see [Continuous deployment for Azure Functions](../functions-continuous-deployment.md).

+ **Ad-hoc deployment**: Deploy from [Visual Studio Code](../functions-develop-vs-code.md#republish-project-files), [Visual Studio](../functions-develop-vs.md#publish-to-azure), or [Azure Functions Core Tools](../functions-run-local.md#project-file-deployment).

+ **Package deployment**: If you don't have the source code, Copilot can help you locate and redeploy the existing deployment package. For more details, see [Get the code deployment package](migrate-plan-consumption-to-flex.md#get-the-code-deployment-package).

> [!CAUTION]
> After deployment, triggers in your new app immediately start processing data from connected services. Review your [data protection strategies](migrate-plan-consumption-to-flex.md#data-protection-strategies) before deploying.

## Verify the migration

After deployment, verify that your new app is running correctly:

```
verify my flex consumption app <APP_NAME> is running correctly
```

Call at least one HTTP trigger endpoint on your new app to confirm it responds as expected.

## Clean up resources

When you're confident the new app works correctly, you can remove the original Consumption plan app. This step is optional — keep the old app as a rollback option for as long as you need.

> [!TIP]
> **No rush here.** The Consumption plan only charges for actual usage, so keeping the old app (with triggers disabled) costs very little.

When you're ready:

```
delete my original consumption app <ORIGINAL_APP_NAME>
```

Copilot always asks for your explicit confirmation before deleting anything.

> [!IMPORTANT]
> Before deleting, make sure you've migrated all functionality, verified no traffic goes to the original app, and backed up any relevant logs or configuration.

## Next step

> [!div class="nextstepaction"]
> [How to use the Flex Consumption plan](../flex-consumption-how-to.md)
