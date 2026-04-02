---
title: "Step 5: Automate Actions in Azure SRE Agent"
description: Create scheduled tasks with email notifications using connectors, subagents, and scheduled task triggers.
ms.topic: tutorial
ms.service: azure-sre-agent
ms.date: 03/30/2026
author: craigshoemaker
ms.author: cshoe
ms.ai-usage: ai-assisted
ms.custom: automation, scheduled tasks, MCP, notifications, getting started
#customer intent: As a site reliability engineer, I want to automate health checks with email notifications so that I can proactively monitor my systems without manual intervention.
---

# Step 5: Automate Actions in Azure SRE Agent

<TimeEstimate minutes={10} /> · Build an automated health check that runs on schedule and sends results via email. You'll connect a tool (Outlook), create a subagent to use it, and attach a scheduled task to trigger it.

## What you'll accomplish

By the end of this step, your agent will:
- Have an Outlook connector that provides email tools
- Have a subagent configured with the SendOutlookEmail tool
- Run a scheduled task that executes the subagent daily

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| **Agent created** | Complete [Step 1](./create-agent.mdx) first |

:::tip Enrich your automated tasks
While not required, completing [Step 2: Add Knowledge](./first-value.mdx) and [Step 3: Connect Source Code](./connect-source-code.mdx) makes your scheduled tasks more effective. Health checks can reference YOUR runbooks and correlate findings to recent code changes—turning generic reports into actionable insights.

---

## Step 1: Add the Outlook connector

First, connect an external tool. You need the connector before you can give its tools to a subagent.

1. Click **Builder** in the left sidebar.
2. Click **Connectors**.
3. Click **Add connector**.
4. Select **Send email (Office 365 Outlook)**.
5. Sign in with your Microsoft account.
6. Click **Add connector**.

![Connectors list showing Outlook connected](./images/connectors-outlook.png)

The connector creates tools your subagents can use: `SendOutlookEmail`, `GetOutlookEmail`, `ListOutlookEmails`, and others.

---

## Step 2: Create a subagent with the email tool

Next, create a subagent that can send emails. Subagents are specialized workers that the agent can invoke for specific tasks.

1. Click **Builder → Subagent builder**.
2. Click **Create subagent** (or the plus icon on the canvas).
3. Name it `email-notifications`.
4. Set **Autonomy** to "Autonomous" (runs without user confirmation).
5. Add the tool: Click the tools dropdown and select **SendOutlookEmail**.
6. Click **Save**.

![Subagent builder showing email-notifications subagent](./images/subagent-builder.png)

Your subagent now appears on the canvas with its connected tool.

---

## Step 3: Add a scheduled task to the subagent

Now link a scheduled task to the subagent. You do this directly from the subagent node.

1. Click the **plus sign (+)** on the left side of the `email-notifications` subagent node.
2. Select **Add scheduled task**.
3. Fill in the task details:

| Field | Value |
|-------|-------|
| **Task name** | `daily-resource-health-report` |
| **Schedule** | Every 24 hours (or use cron: `0 8 * * *` for 8 AM daily) |
| **Notification channel** | (Optional) Teams webhook URL |

4. Enter the task prompt:

```
Check the health of our Azure resources:
1. Verify all container apps are running
2. Check CPU and memory metrics over the last hour
3. Review any recent warning logs
4. Summarize findings and send a report via email using SendOutlookEmail
```

5. Click **Save**.

The canvas now shows the complete workflow: scheduled task → subagent → tool.

<img 
  src={require('./images/canvas-workflow-visualization.png').default} 
  alt="Canvas showing scheduled task connected to subagent with email tool" 
  style={{maxWidth: '500px', margin: '1rem 0'}} 
/>

:::tip Why this order matters
The scheduled task triggers the subagent, which has access to the SendOutlookEmail tool from the Outlook connector. Without the connector, the subagent has no email tool. Without the subagent, the scheduled task has no way to send notifications.

---

## Verify it works

Test your scheduled task:

1. Click **Scheduled tasks** in the left sidebar.
2. Select your task in the list (check the checkbox).
3. Click **Run task now** in the toolbar.
4. Click the chat thread that opens to see execution details.

![Scheduled task execution in chat](./images/scheduled-task-execution.png)

The agent shows:
- Active status and execution time
- Planning and reasoning steps
- Tool invocations (SendOutlookEmail)
- Completion confirmation

---

## What you unlocked

✅ Your agent now:
- Connects to external tools (Outlook)
- Runs subagents with specific tool access
- Executes scheduled tasks automatically
- Sends proactive notifications without manual triggers

🎉 **You've completed the getting started journey!**

---

## What's next?

Your agent is fully operational. Here's where to go deeper:

### Understand the concepts
- [Subagents](subagents.md) — How subagents work, when to use them, autonomy levels
- [Connectors](connectors.md) — All available connectors and how they extend your agent
- [Tools](tools.md) — Built-in tools and how to add custom ones
- [Skills](skills.md) — Modular capabilities your agent loads on demand

### Explore more capabilities
- [Scheduled tasks(scheduled-tasks.md) — Full capability details for automated recurring work
- [Send notifications(send-notifications.md) — Notify your team via Teams, Outlook, or MCP tools
- [Workflow automation(workflow-automation.md) — Chain actions together for complex workflows
- [Agent hooks(agent-hooks.md) — Validate agent responses and audit tool usage
- [Monitor agent usage(monitor-agent-usage.md) — Track how your agent is being used
- [Audit agent actions(audit-agent-actions.md) — Review what your agent did and why

### Add more connectors
- [Tutorial: Connect Azure Data Explorer for log queries](setup-kusto-connector.md)
- [Tutorial: Build custom MCP connectors](setup-mcp-connector.md) — Jira, Slack, Grafana, any API

### Advanced automation
- [Tutorial: Scheduled task patterns](create-scheduled-task.md) — Cron expressions, business hours, chained workflows
- [Tutorial: Set up response plans](setup-response-plan.md) — Configure incident automation
- [Tutorial: Create Python tools](create-python-tool.md) — Extend your agent with custom Python code
- [Tutorial: Agent hooks](agent-hooks.md) — Add guardrails to automated actions
