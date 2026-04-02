---
title: Track Incident Value in Azure SRE Agent
description: Measure your agent's impact with real-time metrics that show which incidents were mitigated, which response plans perform best, and how well your agent meets intent across incidents and scheduled tasks.
ms.topic: concept-article
ms.service: azure-sre-agent
ms.date: 03/30/2026
author: craigshoemaker
ms.author: cshoe
ms.ai-usage: ai-assisted
ms.custom: incidents, metrics, value, daily-reports, incident-scorecard, response-plans, analytics, MTTR, mitigation-rate, intent-met, scheduled-tasks
#customer intent: As an engineering manager, I want to track my agent's incident response impact so that I can prove ROI and optimize my automation strategy.
---

# Track Incident Value in Azure SRE Agent

> [!TIP]
- See your agent's mitigation rate, assist rate, and pending actions — with trend sparklines showing week-over-week change
- Track your agent's Intent Met score — a unified quality metric that measures how well your agent resolves both incidents and scheduled tasks on a 1–5 scale
- Drill into each response plan to identify which automation strategies resolve the most incidents
- Receive automated daily reports covering security findings, resource health, and incident summaries without asking
- Analytics data loads progressively — each card and chart renders as soon as its data is ready, so you see results faster
- Charts auto-rotate date labels and normalize comparison bars for readability at any time range

## The problem: you can't prove your agent is working

You deployed an AI agent to handle incidents. Leadership wants to know: *Is it actually reducing toil? Which incidents is it resolving on its own? Are we getting ROI from this investment?*

Today, answering those questions means manually querying telemetry, cross-referencing incident tickets, and guessing which response plans are effective. There's no single view that shows what your agent did, how well each response plan performed, or whether mitigation rates are improving over time.

Without this data, you can't distinguish a response plan that resolves 80% of incidents autonomously from one that escalates everything to humans. You can't show your team that the agent handled 15 incidents overnight while everyone slept. And you can't make informed decisions about where to invest in better automation.

---

## How incident value tracking works

Your agent records an activity snapshot every time it processes an incident. These snapshots capture the outcome — whether the agent mitigated the incident autonomously, assisted the investigation, or escalated to a human. The **incident metrics** dashboard aggregates these snapshots into five real-time stat cards, a trend chart, and a per-response-plan breakdown.

Navigate to **Monitor → Incident metrics** to see the dashboard.

### Metric cards

Each card shows a count, its proportion relative to total incidents reviewed, and a sparkline with percentage change from the prior period:

| Metric | What it tells you |
|--------|------------------|
| **Incidents reviewed** | Total distinct incidents your agent investigated in the selected time range |
| **Mitigated by agent** | Incidents your agent resolved autonomously — the core ROI metric |
| **Assisted by agent** | Incidents where your agent provided investigation data and the user completed resolution |
| **Mitigated by user** | Incidents resolved entirely by a human — potential automation opportunities |
| **Pending user action** | Incidents waiting for human input — your current backlog |

The **Incident summary** chart below the cards plots all five metrics over time, making it easy to spot trends: is the agent mitigation rate climbing? Is the pending backlog shrinking?

### Chart readability

When a detail trend chart displays more than ten data points — typically when your selected time range spans two or more weeks — the chart automatically rotates x-axis date labels by 45 degrees and increases the bottom margin. This prevents labels from overlapping and keeps dates legible at any time range, whether you're reviewing a 14-day trend or a full month of data. The rotation applies to the trend charts for Hours Saved, Success Rate, Median TTM, and Quality Score.

The **Median Time to Mitigate** comparison card normalizes bar widths as proportions of total resolution time. If your agent resolves incidents with a median of 10 minutes and humans take 30 minutes, the agent bar displays at 25% width and the human bar at 75% — making the speed difference immediately visible without reading the numbers.

### Progressive data loading

The incident metrics dashboard runs its data queries in parallel and renders each result independently. When you open the dashboard or change the time range, individual cards and charts display their data as soon as their backing query completes — you don't wait for every metric to load before seeing results.

In practice:

- **Metric cards** (incidents reviewed, mitigation rate, TTM) typically render first since these are simpler aggregations
- **Charts** (incident summary, root cause, severity distribution) appear as their data arrives
- **Response plan grid** loads last since it computes per-plan metrics across all incidents

When you change the time range or click refresh, previous queries are cancelled and new ones start — each result renders as it arrives.

### Response plan breakdown

Below the chart, a **Response plan** grid breaks down performance per plan. Click any plan name to drill into its individual incident history and root cause categories.

This is where the real decisions happen. You can see at a glance which plans run in [Autonomous mode](run-modes.md) and resolve incidents without human involvement versus plans in Review mode that still require approval. If a plan consistently shows zero autonomous mitigations, it's a signal to either adjust the response plan's instructions or increase its autonomy level.

### Intent Met score

The **Intent Met score** measures how effectively your agent resolves work — whether that's an incident investigation or a scheduled task execution. After each thread completes, an automated evaluation scores the outcome on a 1–5 scale:

| Score | Meaning |
|-------|---------|
| **5** | Exceptionally resolved — exceeded expectations with additional insights |
| **4** | Well resolved — successfully completed with clear evidence of satisfaction |
| **3** | Partially resolved — made progress but didn't fully resolve, or thread is waiting for user action |
| **2** | Poorly resolved — attempted but failed significantly |
| **1** | Completely unresolved — failed to address the core objective |

The Intent Met score card on the **Overview** dashboard displays:

- **Average score** — shown as X/5 across all evaluated threads from the past 30 days
- **Trend sparkline** — daily average scores to track quality over time

The score combines results from both incident threads and scheduled task threads into a single unified metric. Your agent's effectiveness at proactive automation — scheduled health checks, compliance scans, cost monitoring — is measured alongside reactive incident response, giving you one number that captures overall agent quality.

Intent Met scoring is fully automatic. No configuration is needed — every completed incident and scheduled task thread is evaluated using the same scoring criteria. Scheduled tasks that are still waiting for user action receive a score of 3, reflecting their indeterminate outcome.

> [!TIP]
If your Intent Met score is lower than expected, review individual thread conversations in **Monitor → Session insights** to understand where the agent struggled. Common improvements include clearer task instructions, adding relevant connectors, or adjusting the agent's tools.

---

## Before and after

|  | Before | After |
|---|--------|-------|
| **Proving agent value** | Query telemetry, cross-reference tickets, write manual reports | Open one dashboard — mitigation rate, trend, and per-plan breakdown visible instantly |
| **Knowing which plans work** | Guess based on anecdotal feedback | Per-plan grid shows exact mitigation counts alongside autonomy level |
| **Stakeholder updates** | Compile weekly summaries by hand | Daily reports generated automatically with incident counts, health status, and recommended actions |
| **Identifying automation gaps** | No visibility into why incidents escalate | Drill into a response plan to see root cause categories and which incidents the agent couldn't resolve |
| **Measuring scheduled task quality** | Read each task's thread transcript and manually judge whether the objective was met | Automatic Intent Met score evaluates every completed thread — incidents and scheduled tasks combined into a single quality metric |
| **Dashboard loading** | All cards and charts blank until every query finishes | Each card and chart renders independently as its data arrives — start reading metrics in seconds |
| **Reading charts over 30 days** | Date labels overlap and become unreadable — stick to shorter ranges or hover over individual points | Labels auto-rotate at 45 degrees and TTM bars show proportional widths — readable at any time range |

---

## Incident overview

For a real-time view of every incident your agent is handling, go to **Incidents** in the left sidebar.

The page shows summary cards for incident status (Triggered, Acknowledged) and agent investigation status (Pending user input, In progress, Completed), plus a filterable grid of all incidents. Filter by time range, priority, status, investigation status, or search for specific incidents.

Each row links to the agent's investigation thread, so you can review exactly what the agent did — which tools it called, what evidence it found, and what it recommended.

---

## Daily reports

Your agent generates automated daily reports accessible at **Daily reports** in the left sidebar.

Select a date to view that day's report. Each report covers:

- **Security findings** — CVE vulnerabilities across connected repositories, grouped by severity
- **Incidents** — Active, mitigated, and resolved counts with per-incident investigation details
- **Health and performance** — Per-resource health status with availability, CPU, and memory metrics
- **Code optimizations** — Performance recommendations identified by the agent
- **Recommended actions** — Prioritized action items with descriptions and estimated effort

Daily reports replace the "what happened overnight?" morning routine. Instead of asking your agent or querying dashboards, the information is already compiled and waiting.

---

## What makes this different

Incident metrics dashboards aren't new — most observability platforms have them. What's different here is that you're measuring the **agent's** contribution, not just incident volume.

The per-response-plan breakdown answers a question no general-purpose dashboard can: "Which of my AI automation strategies are actually working?" You see the relationship between a plan's autonomy level and its mitigation rate side by side. A plan running in Autonomous mode with high mitigation numbers validates your investment. A plan with all incidents stuck in "Pending user action" tells you the response plan needs better instructions or the agent needs additional tools.

The daily reports go further — they don't just summarize incidents. They correlate security findings, resource health, and performance data into a single view that would otherwise require opening three or four different tools.

The Intent Met score adds a quality dimension that traditional dashboards lack. Instead of just counting incidents resolved, it evaluates **how well** each thread was resolved — using the same criteria for both incidents and scheduled tasks. A scheduled health check that runs but produces empty results scores differently from one that surfaces actionable findings. This gives you a signal to iterate on task instructions, not just monitor completion rates.

---

## Get started

Incident tracking is built-in — open the **Incidents** tab in the agent portal to view scorecards and daily reports once your agent starts handling incidents.

| Resource | What you'll learn |
|----------|-------------------|
| [Set Up a Response Plan →](response-plan.md) | Configure incident response plans that generate tracking data |

## Related capabilities

| Capability | What it adds |
|------------|--------------|
| [Automate Incident Response →(incident-response.md) | Configure response plans that determine how your agent handles each incident type |
| [Automate Tasks on a Schedule →(scheduled-tasks.md) | Set up recurring tasks whose quality is reflected in the Intent Met score |
| [Monitor Agent Usage →(monitor-agent-usage.md) | Track AAU consumption and session insights alongside incident metrics |
| [Audit Agent Actions →(audit-agent-actions.md) | Review the specific actions your agent took during incident investigations |
