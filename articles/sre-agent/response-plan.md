---
title: "Tutorial: Create an Incident Response Plan for Azure SRE Agent"
description: Create a response plan from the Agent Canvas that routes specific incidents to a custom agent, and use the enable or disable toggle to control when it's active.
ms.topic: tutorial
ms.service: azure-sre-agent
ms.date: 03/30/2026
author: craigshoemaker
ms.author: cshoe
ms.ai-usage: ai-assisted
ms.custom: incident trigger, response plan, filter, custom agent, automation, tutorial, toggle, enable, disable, agent canvas
#customer intent: As an SRE, I want to create an incident response plan so that matching incidents are automatically routed to the right custom agent for investigation.
---

# Create an Incident Response Plan (Create Trigger via Agent Canvas) in Azure SRE Agent

:::info What you'll build
A response plan that filters incidents by severity and service, routes matching incidents to a specific custom agent for automated investigation, and a demonstration of the enable/disable toggle. Learn more → [Incident Response Plans(incident-response-plans.md). Time: ~5–10 minutes.

## Prerequisites

- An agent with an incident platform connected (PagerDuty, ServiceNow, or Azure Monitor)
- At least one custom agent configured
- Contributor or Owner role on the agent resource

---

## Step 1: Open the Agent Canvas

In the SRE Agent portal, select your agent. In the left sidebar, go to **Builder** → **Agent Canvas**.

:::warning Delete the default quickstart plan first
When you first connect an incident platform, a default **quickstart** response plan may have been created automatically. Before creating custom plans, switch to **Table view** and select the **Incident response plans** tab to check. Delete the quickstart plan if it exists — overlapping plans can cause incidents to be routed incorrectly or processed twice.

## Step 2: Create a new response plan

In the Agent Canvas, click the **Create** dropdown arrow in the toolbar. Select **Trigger** → **Incident response plan**.

The create dialog opens.

Fill in the filter criteria. The fields shown depend on your incident platform:

- **Incident response plan name** — Enter a descriptive name (e.g., `high-sev-api-trigger`)

For **Azure Monitor**:
- **Severity** — Select one or more severity levels (multiselect)
- **Title contains** (optional) — Add a keyword to narrow matches further

For **PagerDuty / ServiceNow**:
- **Impacted service** — Select the service this plan covers, or select "All"
- **Incident type** — Choose the incident classification, or select "All incident types"
- **Priority** — Select one or more priority levels (multiselect, e.g., P1 and P2)
- **Title contains** (optional) — Add a keyword to narrow matches further

Choose the response configuration:

- **Response custom agent** — Select the custom agent that handles matched incidents
- **Agent autonomy level** — Choose how your agent responds:
  - **Autonomous (Default)** — Your agent independently investigates and performs mitigation
  - **Review** — Your agent proposes actions for your approval before executing

:::note Autonomous mode
When you select **Autonomous (Default)**, an ℹ️ icon appears next to the option. Click it to review the **Autonomous mode acknowledgement** — a summary of what autonomous execution means, including agent boundaries, AI model limitations, and your responsibilities. See [Response Plans → Autonomous mode acknowledgement(incident-response-plans#autonomous-mode-acknowledgement.md) for details.

> [!TIP]
Start with **Review** mode for new plans if you want to validate your agent's investigation behavior before granting full autonomy. New plans default to Autonomous.

**Checkpoint:** All required fields are filled: plan name, impacted service, incident type, and at least one priority level. The **Next** button is enabled.

## Step 3: Preview matching incidents

Click **Next**. The incidents preview shows a table of past incidents that match your filter criteria.

The table displays:
- **Priority**, **Date created**, **Title**, **Incident ID**, and **Status** for each matching incident
- A time range filter (default: Last 90 days) to adjust the preview window

Review the results:
- **Too many matches?** Go back and add a severity restriction or title keyword
- **No matches?** Normal for new services — your plan still works for future incidents
- **Right number?** Your filter is well-tuned

Click **Create incident response plan** to save the plan.

**Checkpoint:** The plan appears in the grid with Status **On** (green badge).

## Step 4: Turn a plan off and on

Select your plan by clicking its checkbox in the grid.

1. Click **Turn off** in the toolbar — a confirmation dialog appears
2. Click **Yes** to disable the plan

The status badge changes to **Off**. The scanner stops matching incidents against this plan. Your filter configuration is preserved.

To re-enable:
1. Select the plan again
2. Click **Turn on** — it takes effect immediately with no confirmation

The status badge returns to **On**.

**Checkpoint:** The toggle works — you can switch a plan between On and Off without deleting it.

## Step 5: Verify in the response plans grid

Your plan is visible right in the **Incident response plans** page grid with the status badge, custom agent, severity filter, and autonomy level columns.

**Checkpoint:** Your plan appears in the grid with the correct status, custom agent, and severity.

:::tip Testing your plan safely
Use the **Title contains** filter to test safely. Set it to match a specific test incident title (e.g., `"[TEST] CPU spike"`) and create a test incident with that title. This validates your agent's behavior without affecting production routing. Once verified, adjust or remove the title filter.

---

## Edit or delete a response plan

### Edit

1. In the response plans grid, click the **plan ID link** to open the plan
2. The edit view opens with all current settings pre-populated
3. Modify the filter criteria, custom agent, or autonomy level
4. Click **Save** to apply changes

### Delete

1. Select the plan using the checkbox in the grid
2. Click **Delete** in the toolbar
3. A confirmation dialog appears — click **Yes** to confirm

Deleted plans stop routing incidents immediately. Active investigations started by the plan continue to completion.

## What you learned

- How to create response plans from the Incident response plans page
- How filter criteria (severity, service, type, title) route incidents to the right custom agent
- How to preview matching historical incidents before committing
- How to use the enable/disable toggle to pause and resume routing
- How to verify plans in the unified grid view in the Agent Canvas
- The difference between Autonomous and Review autonomy levels

## Related

| Resource | What you'll learn |
|----------|-------------------|
| [Incident Response Plans →(incident-response-plans.md) | Understand the full response plans capability |
| [Connect a data source →](setup-kusto-connector.md) | Give your custom agent access to log data |
| [Deep Investigation →](deep-investigation.md) | Complex root cause analysis |
| [Custom agents →](subagents.md) | Specialized custom agents for different incident types |
