---
title: "Tutorial: Connect an ADO Repository with Managed Identity in Azure SRE Agent"
description: Connect an Azure DevOps repository to your agent using managed identity authentication.
ms.topic: tutorial
ms.service: azure-sre-agent
ms.date: 03/30/2026
author: dm-chelupati
ms.author: dchelupati
ms.ai-usage: ai-assisted
ms.custom: managed identity, azure devops, ado, repository, tutorial
---

# Tutorial: Connect an ADO repository with managed identity in Azure SRE Agent

Connect an Azure DevOps repository to your agent using managed identity—no PATs to create or rotate. Your agent uses its own Azure identity to access ADO repos for code-aware investigations.

**Time**: ~10 minutes (including ADO admin setup)

## Prerequisites

- An Azure SRE Agent in **Running** state
- A managed identity enabled on your agent (system-assigned or user-assigned)
- An Azure DevOps organization with at least one repository
- **SRE Agent Administrator** or **Standard User** role on the agent

## Step 1: Grant the managed identity access to your ADO organization

Before connecting from the agent portal, your managed identity must have access to the Azure DevOps organization.

1. Go to your [Azure DevOps organization settings](https://dev.azure.com/) and select your organization.
2. Navigate to **Organization settings** > **Users**.
3. Select **Add users**.
4. Search for your agent's managed identity by its service principal name or object ID.
5. Set the access level to **Basic** (or higher).
6. Add the identity to projects with **Code (Read)** permissions on the target repositories.

**Checkpoint:** The managed identity appears in the ADO Users list with a Basic access level.

## Step 2: Navigate to Knowledge sources

1. Open your agent in the [Azure SRE Agent portal](https://sre.azure.com).
2. In the left sidebar, expand **Builder**.
3. Select **Knowledge sources**.

**Checkpoint:** The Knowledge Sources page loads showing any existing repository connections.

## Step 3: Open the Add Repository dialog

Select **Add repository**.

**Checkpoint:** The Add repositories dialog opens showing platform selection cards (GitHub, Azure DevOps).

## Step 4: Select Azure DevOps with Managed Identity

1. Select the **Azure DevOps** platform card.
2. Under **Sign In Methods**, select **Managed Identity**.

**Checkpoint:** The managed identity configuration form appears with an organization field and identity dropdown.

## Step 5: Configure the managed identity connection

1. Enter your Azure DevOps **Organization** name—the part after `dev.azure.com/` in your ADO URL.
2. From the managed identity dropdown, select your identity:
   - **System assigned**—uses the agent's built-in identity
   - **User assigned**—select a specific identity attached to the agent
3. Select **Connect**.

**Checkpoint:** The button changes to **Connected** with a checkmark.

> [!NOTE]
> If the dropdown is empty, your agent might not have a managed identity enabled. Select the **Add identity** link below the dropdown to open the Azure portal Identity blade for your agent resource.

## Step 6: Advance to repository selection

Select **Next** to proceed to the repository selection step.

**Checkpoint:** The dialog advances to show a project picker and repository grid.

## Step 7: Select a project and add repositories

1. From the **Azure DevOps Project** dropdown, select the project containing your repositories.
2. Select **Add** to add a repository row.
3. From the **Repository** dropdown, select a repository from the project.
4. Enter a **Display name** for the repository.
5. Optionally enter a **Description**.
6. Repeat for more repositories.
7. Select **Save**.

**Checkpoint:** Selected repositories appear in the Knowledge Sources page.

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Identity dropdown is empty | Agent has no managed identity enabled | Enable a system-assigned identity or attach a user-assigned identity in the Azure portal |
| **Connect** button fails | Organization name is missing | Enter the ADO organization name before connecting |
| Repos don't load after connecting | MI doesn't have access to the ADO organization | Add the MI service principal as a user in ADO Organization Settings > Users |
| FIC connection fails | FederatedClientId and FederatedTenantId not both provided | Both fields are required when using FIC—provide both or neither |

## Related content

- [Managed identity for ADO repos](managed-identity-ado-repos.md)
- [Connect knowledge sources](connect-knowledge.md)
- [Set up Azure DevOps connector](azure-devops-connector.md)
