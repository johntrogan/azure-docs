---
title: Schedule tasks with Azure SRE Agent
description: Learn how to use scheduled tasks in SRE Agent to automate monitoring, enforce security, and validate recovery.
author: craigshoemaker
ms.topic: overview
ms.date: 03/30/2026
ms.author: cshoe
ms.service: azure-sre-agent
---

# Scheduled Tasks in Azure SRE Agent

<video 
  controls 
  style={{width: '100%', maxWidth: '800px', marginTop: '1rem', marginBottom: '1.5rem', borderRadius: '8px'}}
>
  <source src={useBaseUrl('/video/Scheduled_Tasks.mp4')} type="video/mp4" />
  Your browser does not support the video tag.
</video>

> [!TIP]
- Issues caught before users notice — proactive monitoring replaces reactive dashboards
- Correlated insights instead of raw metrics — your agent reasons across data sources
- Describe checks in natural language — no scripts to write or maintain
- Create, edit, and manage tasks from the portal or chat

## The problem

Operational tasks repeat. Every morning someone checks resource health. Every Monday someone pulls cost data. Every hour someone scans for anomalies. These repetitive tasks consume your team's time with predictable, automatable work — time better spent investigating real issues.

Traditional monitoring compounds the problem. Alert rules fire *after* a threshold is breached — by the time you see it, users are already affected. Dashboards show raw data but don't explain what it means. Each alert is isolated: your CPU alert doesn't know about the deployment that happened ten minutes ago. You correlate manually, across tools, every single time.

## How scheduled tasks work

<DiagramSvg title="Scheduled task execution flow" className="diagram" />

Your agent runs tasks on a schedule you define. Describe what you want done in natural language, set the frequency, and your agent handles execution automatically. Each execution creates a conversation thread where the agent plans its approach, queries data sources, reasons about findings, and produces an actionable summary.

This isn't a cron job running a script. Your agent uses its [connectors](connectors.md), [tools](tools.md), [knowledge](memory.md), and [memory](memory.md) to understand context. It notices that error rates are trending up 15% day-over-day even though they haven't hit the alerting threshold. It catches that storage usage will hit quota in three days at the current growth rate. It connects yesterday's deployment to today's exceptions.

Select **Scheduled tasks** in the left sidebar to manage all your tasks.

## What makes this different

|  | Alert rules | Dashboards | Cron jobs | Scheduled tasks |
|--|------------|-----------|-----------|-----------------|
| **When** | After threshold breached | When you look | On schedule | On your schedule, before thresholds |
| **What it shows** | Single metric | Raw data | Script output | Correlated findings with explanation |
| **Context** | None | Whatever you configured | What the script queries | Cross-source, compared to baseline |
| **Action** | You investigate | You investigate | Whatever the script does | Summary with recommended next steps |
| **Adapts** | Static rules | Static views | Static scripts | [Memory](memory.md) captures patterns over time |

Unlike cron jobs, your agent understands natural language. You don't write scripts — you describe what needs to happen. Unlike runbooks, scheduled tasks execute automatically with the autonomy level you choose.

## Before and after

| Before | After |
|--------|-------|
| Check dashboards manually every morning | Agent checks proactively and posts summary |
| Correlate alerts across tools by hand | Agent correlates across all connected sources |
| Issues discovered after users report them | Trends caught before they become incidents |
| Write and maintain monitoring scripts | Describe checks in natural language |
| Each team member checks differently | Consistent automated checks every time |
| Need to change a task? Delete and recreate | Edit any task in place — execution history preserved |

## Task dashboard

<Screenshot
  src="/img/screenshots/portal-scheduled-tasks.png"
  alt="Scheduled tasks dashboard showing task list, metrics, and toolbar actions"
/>

The dashboard displays three key metrics at the top:

| Metric | Description |
|--------|-------------|
| **Active tasks** | Tasks currently enabled and running on schedule |
| **Total tasks** | All tasks including paused and completed |
| **Total runs** | Completed executions across all tasks |

The task list shows each task with sortable columns:

| Column | Description |
|--------|-------------|
| **Name** | Task identifier — click to view execution history |
| **Task status** | On, Off, Ended, or Failed |
| **Schedule** | Human-readable format (e.g., "Daily at 8:00 AM") |
| **Created by** | Who created the task |
| **Last run** | Most recent execution time |
| **Next run** | Upcoming scheduled execution |
| **Completed runs** | Total successful executions |

## Editing a task

Modify any scheduled task directly — change the schedule, update instructions, reassign the subagent, or adjust run parameters. Your task's execution history is preserved.

### Three ways to edit

| Method | Steps |
|--------|-------|
| **Toolbar** | Select a task checkbox → click **Edit task** in the toolbar |
| **Row menu** | Click **⋯** on any task row → select **Edit task** |
| **Execution view** | Click a task name to open execution history → click **Edit task** |

<Screenshot
  src="/img/screenshots/portal-scheduled-task-context-menu.png"
  alt="Row context menu showing Edit task, Turn off, Run task now, and Delete options"
/>

The edit dialog opens with all current values pre-populated. Change any combination of fields:

- **Task name** and **instructions** — update what the agent does
- **Schedule** — change frequency, time, or switch to a custom cron expression
- **Response subagent** — reassign to a different subagent
- **Date range** — adjust start date or set a new end date
- **Message grouping for updates** — switch between same thread or new threads per run
- **Set a run limit** — add, change, or remove the maximum execution count
- **Agent autonomy level** — switch between Autonomous and Review mode. When you select **Autonomous**, an info icon (ℹ️) appears — click it to review the **Autonomous mode acknowledgement**, which explains agent boundaries, AI model limitations, your responsibilities, and liability terms.

Click **Save** to apply your changes.

> [!NOTE]
**Save** is disabled until you modify at least one field, preventing accidental no-op updates.

## Example use cases

| Use case | What the agent does |
|----------|-------------------|
| **Daily health check** | Reviews resource health, checks for degraded services, reports findings |
| **Cost anomaly detection** | Compares spend to baselines, flags unexpected increases |
| **Security posture review** | Checks for misconfigurations, expired certificates, open ports |
| **Deployment verification** | Verifies recent deployments are healthy after rollout |
| **SLA reporting** | Generates weekly availability and performance summaries |

### Example task prompts

**Daily health check:**
> Review the health of all container apps in resource group prod-apps. Report any with restarts in the last 24 hours, memory usage above 80%, or error rates above 1%. Compare current error rates to last week's average.

**Cost anomaly detection:**
> Analyze Azure cost data for my subscription. Compare today's spending rate to the 7-day average. Flag any resource group where spending increased more than 20%.

## Get started

| Resource | What you'll learn |
|----------|-------------------|
| [Create a Scheduled Task →](create-scheduled-task.md) | Step-by-step tutorial |

## Related capabilities

| Capability | What it adds |
|------------|--------------|
| [Execute Mitigations →(execute-mitigations.md) | Take action when monitoring detects issues |
| [Workflow Automation →(workflow-automation.md) | Chain tasks with triggers, custom agents, and notifications |
| [Send Notifications →(send-notifications.md) | How the agent delivers findings to your team |
| [Run Modes →](run-modes.md) | Control agent autonomy per task |
| [Connectors →](connectors.md) | Access third-party observability tools |
