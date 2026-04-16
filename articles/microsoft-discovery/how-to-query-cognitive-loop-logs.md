---
title: Query CogLoop logs for Microsoft Discovery
description: Learn how to query CogLoop AI orchestration logs for a Microsoft Discovery workspace in the Log Analytics workspace within the Managed Resource Group, to investigate investigation progress, errors, and decision-making behavior.
author: mukesh-dua
ms.author: mukeshdua
ms.service: azure
ms.topic: how-to
ms.date: 04/15/2026

#CustomerIntent: As a Discovery platform developer or administrator, I want to query CogLoop logs so that I can monitor investigation progress, diagnose stalls, and investigate orchestration errors.
---

# Query CogLoop logs for Microsoft Discovery

Microsoft Discovery CogLoop is the AI orchestration engine that drives investigation progress. It continuously runs two subloops - **Act** and **Cognition** to plan and execute research tasks on your behalf. CogLoop logs are automatically stored in the `DiscoveryCogLoopLogs` table in the Log Analytics workspace inside the workspace's Managed Resource Group (MRG).

> [!IMPORTANT]
> `DiscoveryCogLoopLogs` is an **Auxiliary** tier table. Each query must target this table only (cross-table joins aren't supported).

This article explains how to query CogLoop logs to monitor investigation activity, diagnose stalls, and investigate orchestration errors.

## Prerequisites

Before you begin, navigate to the Log Analytics workspace in your workspace's MRG. For instructions, see [Access resource logs for Microsoft Discovery resources](how-to-access-resource-logs.md).

## Key identifiers

Before writing queries, familiarize yourself with the following identifiers:

| Identifier | Description | Where to find it |
|---|---|---|
| `InstanceId` | Unique ID for an investigation's CogLoop instance, in the format `cog:{project}:{investigation}-{shortHash}` | Construct from your project name and investigation name in Discovery Studio, for example, `cog:testworkspace:investigation1-436edc` |
| `ModuleName` | Which subloop generated the log: `Act` or `Cognition` | Extracted from the `Properties` JSON field |
| `CorrelationId` | W3C trace ID, present on user-triggered operations | Returned in API responses |

## Query CogLoop logs

After opening the Logs interface in the workspace's MRG Log Analytics workspace:

1. In the left panel, select the **Tables** tab.
2. Expand **Custom Logs** and locate **`DiscoveryCogLoopLogs`**.
3. Select **Run** next to `DiscoveryCogLoopLogs` to execute a default query and confirm logs are being ingested.

:::image type="content" source="media/how-to-query-workspace-logs/open-log-analytics-table.png" alt-text="Screenshot of the Log Analytics Tables panel with the DiscoveryCogLoopLogs_CL table selected in the Custom Logs section." lightbox="media/how-to-query-workspace-logs/open-log-analytics-table.png":::

## Log schema

The `DiscoveryCogLoopLogs` table includes the following key fields:

| Field | Description |
|---|---|
| `TimeGenerated` | Timestamp when the log entry was ingested into Log Analytics |
| `Message` | Primary log message content |
| `LogLevel` | Log severity level: `Information`, `Warning`, `Error` |
| `CorrelationId` | Unique identifier linking all log entries for a specific user-triggered operation |
| `Exception` | Exception details for error entries |

## Example queries

### View recent logs

```kql
DiscoveryCogLoopLogs
| take 100
```

### Filter by time range

```kql
DiscoveryCogLoopLogs
| where TimeGenerated > ago(1h)
| order by TimeGenerated desc
```

### View all errors and warnings

Quick sweep of all CogLoop errors and warnings, useful to confirm whether there were any issues during a specific investigation session.

```kql
DiscoveryCogLoopLogs
| where TimeGenerated > ago(1h)
| where LogLevel in ("Error", "Warning")
| project TimeGenerated, LogLevel, Message
| order by TimeGenerated desc
```

Adjust `ago(1h)` to `ago(6h)`, `ago(1d)`, and so on, as needed.

### Check whether a CogLoop instance is running

Verify whether CogLoop is actively monitoring your investigation. Replace `<your-project>` with your project name.

```kql
DiscoveryCogLoopLogs
| where TimeGenerated > ago(15m)
| where Message has "IsInstanceRunningAsync"
| extend InstanceId = extract(@"InstanceId: (cog:[^\s]+)", 1, Message)
| where InstanceId has "<your-project>"
| summarize last_checked = max(TimeGenerated) by InstanceId
| order by last_checked desc
```

A result with a `last_checked` within the past few minutes indicates the instance is actively being polled. No results means CogLoop isn't monitoring the investigation.

### List all active investigations being monitored

See which investigations CogLoop is currently tracking, useful when you want to confirm whether your investigation is in scope.

```kql
DiscoveryCogLoopLogs
| where TimeGenerated > ago(30m)
| where Message has "IsInstanceRunningAsync"
| extend InstanceId = extract(@"InstanceId: (cog:[^\s]+)", 1, Message)
| where isnotempty(InstanceId)
| summarize last_seen = max(TimeGenerated) by InstanceId
| order by last_seen desc
```

### View all activity for a specific investigation

Retrieve the complete CogLoop activity log for one investigation. Replace `cog:<your-project>:<your-investigation>` with the full `InstanceId` prefix, for example, `cog:fabforbook:leukemia-2`.

```kql
DiscoveryCogLoopLogs
| where Message has "cog:<your-project>:<your-investigation>"
| project TimeGenerated, LogLevel, Message
| order by TimeGenerated asc
```

### View errors for a specific investigation

View all errors and warnings for a single investigation instance.

```kql
DiscoveryCogLoopLogs
| where LogLevel in ("Error", "Warning")
| where Message has "cog:<your-project>:<your-investigation>"
    or Exception has "cog:<your-project>:<your-investigation>"
| project TimeGenerated, LogLevel, Message, Exception
| order by TimeGenerated asc
```

### Diagnose why CogLoop isn't making progress

CogLoop enters a wait loop when it's blocked, typically waiting for a running task to complete or for human intervention on a flagged task.

```kql
DiscoveryCogLoopLogs
| where Message has_any ("No task updates", "Skipping redundant action", "Cognition-Wait")
| extend InstanceId = extract(@"InstanceId: (cog:[^\s]+)", 1, Message)
| extend SleepTime = toint(extract(@"wait period of (\d+) seconds", 1, Message))
| project TimeGenerated, LogLevel, InstanceId, SleepTime, Message
| order by TimeGenerated desc
```

What to look for in the results:

| Pattern | Meaning |
|---|---|
| Repeated `No task updates during wait period of 60 seconds` | CogLoop is waiting for a running task to finish; check the associated Act task for status |
| `Skipping redundant action: Cognition-Wait with result: NO UPDATE` | Cognition is also blocked; no new decisions are being made |
| Pattern persists for hours | A task may be `FLAGGED_HUMAN` and requires your review in Discovery Studio |

### View CogLoop decision log for an investigation

View the reasoning behind CogLoop's recent decisions, which tool it chose and why. Useful to understand why CogLoop is repeatedly waiting instead of making progress.

```kql
DiscoveryCogLoopLogs
| where Message has "PickBest"
| where Message has "cog:<your-project>:<your-investigation>"
    or CorrelationId == "<your-correlationId>"
| extend chosen_tool = extract(@'"ChosenTool":\s*"([^"]+)"', 1, Message)
| extend confidence = toint(extract(@'"Confidence":\s*(\d+)', 1, Message))
| extend reasoning = extract(@'"Reasoning":\s*"([^"]+)"', 1, Message)
| project TimeGenerated, chosen_tool, confidence, reasoning
| order by TimeGenerated desc
```

The `reasoning` field explains exactly why CogLoop chose to wait or act, this is the most useful field for understanding investigation stalls.

### Detect Act and Cognition subloop errors

The two CogLoop subloops can fail independently. This query breaks down errors by subloop to identify which one is failing.

```kql
DiscoveryCogLoopLogs
| where LogLevel == "Error"
| where Message has_any ("Act loop error", "Cognition loop error")
| extend ModuleName = case(
    Message has "Act loop", "Act",
    Message has "Cognition loop", "Cognition",
    "Unknown"
  )
| summarize error_count = count() by ModuleName, bin(TimeGenerated, 1h)
| order by TimeGenerated desc
```

- **Act loop errors** affect task execution and tool invocation.
- **Cognition loop errors** affect planning and decision-making.
- Errors in both loops simultaneously may indicate a systemic issue such as a service outage or network partition.

### Detect context window saturation

When CogLoop's context window exceeds 80% capacity, it attempts to reset working memory. If that reset also fails because the context is too large, an error is logged.

```kql
DiscoveryCogLoopLogs
| where Message has_any ("Context window saturation", "working memory context reset")
    or (LogLevel == "Error" and Message == "Error during working memory context reset")
| extend event_type = case(
    Message has "80%", "Warning: Context window near full",
    Message == "Error during working memory context reset", "Error: Reset failed",
    "Other"
  )
| project TimeGenerated, LogLevel, event_type, Message, Exception
| order by TimeGenerated asc
```

> [!NOTE]
> This error indicates the investigation has accumulated a large context. CogLoop attempted to compress it but the summary was too large for the model's input limit. If this error repeats without recovery, the investigation may be stuck. Contact your Discovery administrator.

### Detect circuit breaker events

A circuit breaker opening means that repeated LLM API call failures have caused CogLoop to pause sending requests temporarily to prevent cascading failures.

```kql
DiscoveryCogLoopLogs
| where LogLevel == "Error"
| where Exception has "BrokenCircuitException"
| project TimeGenerated, LogLevel, Message, Exception
| order by TimeGenerated desc
```

> [!NOTE]
> Circuit breaker events typically self-heal when the circuit resets after a cooldown period. If they persist, contact Microsoft Support.

## Troubleshooting

### No data in DiscoveryCogLoopLogs

| Cause | Resolution |
|---|---|
| Workspace was recently created and no investigations have run | Run an investigation to generate log entries |
| Time range is too narrow | Expand the time range to the last 24 hours |
| Logs are delayed due to ingestion latency | Wait a few seconds and rerun the query |

### Investigation appears stuck with no progress

| Cause | Resolution |
|---|---|
| A task is in `FLAGGED_HUMAN` state | Open the investigation in Discovery Studio and address the flagged task |
| CogLoop is waiting for a running tool task to complete | Check Supercomputer logs for the associated job. See [Query supercomputer logs](how-to-query-supercomputer-logs.md) |
| Context window saturation reset failed | Contact your Discovery administrator |

### Query timeout or slow performance

| Cause | Resolution |
|---|---|
| Query scope is too broad | Narrow the time range or filter by `InstanceId` |
| Large dataset with no filters | Add filters on `LogLevel`, `InstanceId`, or `Message` before aggregating |

## Related content

- [Observability in Microsoft Discovery](concept-observability.md)
- [Access resource logs for Microsoft Discovery resources](how-to-access-resource-logs.md)
- [Query workspace logs](how-to-query-workspace-logs.md)
- [Query supercomputer logs](how-to-query-supercomputer-logs.md)
- [View activity logs for Microsoft Discovery resources](how-to-view-activity-logs.md)
