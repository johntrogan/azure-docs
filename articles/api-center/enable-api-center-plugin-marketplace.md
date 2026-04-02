---
title: Enable API Center Plugin Marketplace     - Azure API Center
description: Enable a plugin marketplace endpoint for your Azure API center. Developers can configure it in GitHub Copilot or Claude Code to discover and install plugins from your inventory.

ms.service: azure-api-center
ms.topic: how-to
ms.date: 04/02/2026
 
ms.custom: 
# Customer intent: As an API program manager, I want to create a plugin marketplace from my API center so AI developers can find and install plugins in my inventory.
---

# Enable a marketplace for API center plugins

<!-- Is this a preview? -->
<!-- What can customer expect plugins to consist of? MCP servers? Skills? etc. -->
<!-- Permissions/auth? -->
<!-- Sync? -->

This article shows how to enable and consume a plugin marketplace endpoint in [Azure API Center](overview.md). The plugin marketplace endpoint uses the API Center data plane API to catalog the AI plugins available in the API center inventory. 

After you configure the plugin marketplace, developers can add it to their GitHub Copilot CLI or Claude Code development environment to discover and install plugins from your API center.

## Prerequisites

* An API center in your Azure subscription. If you don't have one, see [Quickstart: Create your API center](set-up-api-center.md).

* The API center portal enabled and set up for your API center. For details, see [Set up and customize your API Center portal](set-up-api-center-portal.md). 

* [GitHub Copilot CLI](https://github.com/github/copilot-cli) or [Claude Code](https://www.anthropic.com/claude) installed in your development environment.


## Confirm plugin marketplace endpoint is enabled for your API center

After setting up the API Center portal, confirm that the plugin marketplace endpoint is enabled for your API center by cloning it locally. 

> [!TIP]
> After setting up the API center portal, it can take several minutes for the plugin marketplace endpoint to be available. 

The marketplace endpoint is of the following form:

```
https://<service name>.data.<region>.azure-apicenter.ms/workspaces/default/plugins/marketplace.git
```

To clone it, use a command similar to the following in your terminal, replacing the service name and region with the values from your API center:

```bash
git clone https://myapicenter.data.eastus.azure-apicenter.ms/workspaces/default/plugins/marketplace.git
```

The `marketplace` folder of the cloned repository contains folders for marketplace configuration in Claude Code and GitHub Copilot CLI, and folders for each plugin in your API center inventory. For example:

```
marketplace/
    .claude-plugin/
    .github/
    plugins/
        plugin1/
        plugin2/
        ...
```

Each plugin folder contains JSON files with the plugin metadata and configuration.

## Add plugin marketplace to GitHub Copilot CLI 

Developers can add the plugin marketplace from your API center's marketplace endpoint to GitHub Copilot CLI by using the `marketplace add` command. For example, add it by using the GitHub Copilot CLI in interactive mode with a command similar to the following. Replace the service name and region with the values from your API center:

```bash
copilot -i "marketplace add https://myapicenter.data.eastus.azure-apicenter.ms/workspaces/default/plugins/marketplace.git"
```

Follow the prompts to add the plugin marketplace to your GitHub Copilot CLI. 

Once the marketplace is added, use the `copilot marketplace list` command to see the plugins from your API center inventory. 


For information about installing plugins from the marketplace in GitHub Copilot CLI, see [GitHub Copilot CLI documentation](https://docs.github.com/copilot/how-tos/copilot-cli/customize-copilot/plugins-finding-installing).

## Add plugin marketplace to Claude Code

Developers can add the plugin marketplace from their API center's marketplace endpoint by using the `marketplace add` command. For example, use Claude Code in chat mode with a command similar to the following command. Replace the service name and region with the values from your API center:

```bash
/plugin marketplace add https://myapicenter.data.eastus.azure-apicenter.ms/workspaces/default/plugins/marketplace.git
```

Follow the prompts to add the plugin marketplace.

After you add the marketplace, use the `/plugin marketplace list` command to see the plugins from your API center inventory. 

For information about installing plugins from the marketplace in Claude Code, see [Claude Code documentation](https://code.claude.com/docs/en/discover-plugins).

## Related content

* [Set up and customize your API Center portal](set-up-api-center-portal.md)
* [Discover and consume APIs - VS Code extension](discover-apis-vscode-extension.md)


