---
title: Connect source code to Azure SRE Agent
description: Connect a GitHub or Azure DevOps repository so your agent performs root cause analysis and correlates production issues to specific code changes.
ms.topic: tutorial
ms.service: azure-sre-agent
ms.date: 03/30/2026
author: craigshoemaker
ms.author: cshoe
ms.ai-usage: ai-assisted
ms.custom: source code, github, azure devops, root cause analysis, connectors, getting started
#customer intent: As a site reliability engineer, I want to connect my source code repository so that my agent can correlate production issues to specific code changes during investigations.
---

# Step 3: Connect Source Code in Azure SRE Agent

Connect your GitHub or Azure DevOps repository. Your agent can now perform **root cause analysis**—correlating production issues to specific code.

## What you'll accomplish

By the end of this step, your agent will:
- Analyze source code during investigations
- Provide **file:line references** for issues
- Create To-Do Plans showing investigation steps
- Correlate production symptoms to code changes

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| **Agent created** | Complete [Step 1](overview.md) first |
| **GitHub or Azure DevOps account** | Access to the repositories you want to connect |

---

## Choose your authentication method

| Method | When to use |
|--------|-------------|
| **OAuth** | Sign in with your GitHub account — no token needed, easiest setup |
| **PAT** | Provide a Personal Access Token with `repo` scope — works for orgs with SSO restrictions |

---

## Connect your repository

Connect a GitHub repository so your agent indexes it as a knowledge source. The dialog shows a browsable list of your repositories — select from the dropdown instead of typing URLs manually.

### Step 1: Open the Add Repository dialog

During onboarding, select the **Add repository** card in the Knowledge Base step.

For an existing agent, go to **Builder** → **Knowledge settings** and select the **Add repository** action card.

### Step 2: Choose a platform

1. Select **GitHub** or **Azure DevOps**.
2. Choose your sign-in method:

| Method | When to use |
|--------|-------------|
| **Auth** (OAuth) | Sign in with your GitHub or Azure DevOps account — no token needed |
| **PAT** | Provide a Personal Access Token with `repo` scope |

3. Complete authentication:
   - **OAuth:** Click **Sign in to GitHub** (or **Sign in to Azure DevOps**) and complete the authentication popup.
   - **PAT:** Enter your token in the **Provide PAT** field and click **Connect**.

:::note Popup blocker?
If the sign-in dialog doesn't appear, check that your browser isn't blocking popups from `sre.azure.com`.

4. On success, a **Connected** card appears showing your authenticated account.

5. Click **Next**.

### Step 3: Select repositories

After authentication, the **Repository URL** field shows a dropdown of your repositories:

- **GitHub repos** appear as `org/repo-name`, sorted alphabetically (up to 100 repos)
- **Azure DevOps repos** appear after selecting a project from the **Azure DevOps Project** dropdown, sorted alphabetically

Select a repository from the dropdown — the **Display name** auto-fills with the repository name. You can also type any valid repository URL directly into the field.

To add multiple repositories, click **Add** to insert additional rows.

:::tip Freeform entry
The dropdown allows freeform typing. If your repository doesn't appear in the list (for example, if you have more than 100 repos), type the full URL directly.

### Step 4: Confirm and save

Click **Add repository** to save.

The system automatically creates the appropriate GitHub OAuth or Azure DevOps OAuth connector if one doesn't already exist.

### Managing connected repositories

When you reopen the Add Repository dialog, existing connected repositories appear as read-only rows in the grid.

**To remove a repository:**

1. Open **Builder** → **Knowledge settings** → click the **Add repository** action card
2. Find the repository row in the grid
3. Click the **trash icon** on the row to mark it for deletion
4. Click **Add repository** to save changes
5. A **Confirm changes** dialog appears listing the repositories that will be removed
6. Click **Confirm** to proceed or **Cancel** to keep them

**To update authentication:** If your PAT expires or you need to switch accounts, re-open the Add Repository dialog and re-authenticate with new credentials.

---

## Alternative: MCP + custom agent

For full GitHub API access — search code, read files, list commits across all repositories — connect GitHub as an MCP server with a dedicated custom agent.

This approach uses the Model Context Protocol (MCP) to connect GitHub tools to a custom agent. Follow the step-by-step tutorial:

→ [Set Up MCP Connector](mcp-connector.md) — connects GitHub MCP, creates a custom agent, and tests it in chat (~10 minutes)

---

## What you learned

- Your agent now analyzes source code during investigations
- It provides file and line references for issues
- It creates To-Do Plans showing investigation steps
- It correlates production symptoms to code changes

---

## Related

| Resource | What you'll do |
|----------|-------------------|
| [Root cause analysis(root-cause-analysis.md) | How your agent uses source code to find root causes |
| [Deep investigation(deep-investigation.md) | Extended multi-hypothesis analysis using connected repos |
| [Agent Playground(agent-playground.md) | Test MCP tools and custom agents interactively |
| [Custom agents](sub-agents.md) | How custom agents extend your agent's capabilities |
| [Connectors](connectors.md) | All connector types and how they work |
