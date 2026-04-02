---
title: Set up PagerDuty incident indexing in Azure SRE Agent
description: Connect PagerDuty to Azure SRE Agent so your agent automatically picks up, investigates, and resolves PagerDuty incidents.
ms.topic: tutorial
ms.service: azure-sre-agent
ms.date: 03/30/2026
author: craigshoemaker
ms.author: cshoe
ms.ai-usage: ai-assisted
ms.custom: pagerduty, incident-indexing, setup, tutorial
#customer intent: As an SRE, I want to set up PagerDuty incident indexing so that my agent automatically picks up and investigates PagerDuty incidents.
---

# Set up PagerDuty incident indexing in Azure SRE Agent

Connect your PagerDuty account so your agent automatically picks up PagerDuty incidents and investigates them.

**Time**: ~5 minutes

## Prerequisites

- An Azure SRE Agent in **Running** state
- A PagerDuty account with at least one service configured
- A PagerDuty REST API v2 access key ([PagerDuty API Access Keys](https://support.pagerduty.com/docs/api-access-keys))
- **Contributor** role or higher on the SRE Agent resource

## Step 1: Open Incident platform

1. Navigate to [sre.azure.com](https://sre.azure.com) and select your agent.
2. In the sidebar, expand **Builder**.
3. Select **Incident platform**.

**Checkpoint:** You see the "Incident platform" heading with a platform dropdown.

## Step 2: Select PagerDuty

1. Select the **Incident platform** dropdown.
2. Select **PagerDuty**.

If another platform is already connected, you'll be prompted to disconnect it first. Your agent connects to one incident platform at a time.

**Checkpoint:** The PagerDuty form appears with a green heading and an API key field.

## Step 3: Enter your PagerDuty API key

Enter your **REST API access key** in the field.

The agent validates your key instantly by calling the PagerDuty API.

> [!TIP]
> In PagerDuty, go to **Integrations > API Access Keys > Create New API Key**. The key needs incident read access at minimum. For full functionality (acknowledge, resolve, add notes), use a key with write access.

**Checkpoint:** No validation error appears below the key field. The Save button becomes active.

If you see "The access key is not valid. Please try again."—verify your key in PagerDuty under Integrations > API Access Keys. Regenerate the key if it has expired.

## Step 4: Enable quickstart response plan

The **Quickstart response plan** checkbox is enabled by default. This creates a response plan that:

- Matches P1 priority PagerDuty incidents
- Runs in fully autonomous mode
- Covers all impacted services

Leave this checked to start with automated P1 incident handling. If you uncheck it, no incidents appear until you create a response plan manually.

## Step 5: Save and connect

1. Select **Save**.
2. Wait for the connectivity indicator to show **"PagerDuty is connected."**—typically 30 seconds to 2 minutes.

> [!WARNING]
> If you see "Connection to PagerDuty failed," verify your API key is correct, hasn't expired, and has incident read permissions.

**Checkpoint:** The connectivity indicator shows a green "PagerDuty is connected." status.

## Step 6: Verify incidents appear

1. In the sidebar, select **Activities**.
2. Select **Incidents**.
3. Wait 1-2 minutes for the scanner to complete its first cycle.

PagerDuty incidents matching your response plan filters appear in the incident overview grid.

**Checkpoint:** PagerDuty incidents appear with columns for ID, Title, Priority, Status, Agent Status, Created, Impacted Service, and Handler.

## Edit or disconnect PagerDuty

### Edit connection settings

1. Go to **Builder** > **Incident platform**.
2. Select **Edit**.
3. Modify the API key or other settings.
4. Select **Save**.

### Disconnect PagerDuty

1. Go to **Builder** > **Incident platform**.
2. Select **Disconnect**.
3. Confirm the disconnection.

After disconnecting, the agent stops scanning for PagerDuty incidents. Your response plans are preserved and reactivate when you reconnect.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "The access key is not valid" | Verify the key in PagerDuty > Integrations > API Access Keys |
| "Connection to PagerDuty failed" | Check key permissions—needs incident read access at minimum |
| No incidents appear after setup | Wait 1-2 minutes for the first scan cycle |
| Need to switch from another platform | Disconnect the current platform first |
| Connectivity indicator stays "Connecting" | Allow up to 2 minutes. If it times out, re-enter the API key |

## Related content

- [PagerDuty incident indexing](pagerduty-incidents.md)
- [Incident response plans](incident-response-plans.md)
- [Incident response](incident-response.md)
