---
title: Durable task for AI agents - Azure
description: Learn how durable execution on Azure provides fault-tolerant, scalable infrastructure for production AI agents using Durable Functions, Durable Task SDKs, and the Durable Task Scheduler.
author: greenie-msft
ms.topic: conceptual
ms.date: 03/15/2026
ms.author: nigreenf
---

# Durable task for AI agents

Production AI agents are distributed systems. They call LLMs that can be slow or rate-limited, invoke external tools and APIs that may fail transiently, maintain conversation state across sessions that span hours or weeks, and need to scale across compute instances to handle variable demand. These are the same reliability and coordination challenges that distributed cloud services have faced for years, and they require the same kinds of solutions.

[Durable task on Azure](./index.yml) ([Durable Functions](./durable-functions-overview.md), the [Durable Task SDKs](./durable-task-scheduler/durable-task-scheduler.md), and the [Durable Task Scheduler](./durable-task-scheduler/durable-task-scheduler.md) as the managed backend) provides **durable execution** for Azure: a fault-tolerant approach to running code that automatically handles failures, automatic checkpointing, and distributed coordination. Instead of writing plumbing code for retries, checkpointing state, and error recovery, you offload that complexity to durable task and focus on the business logic that differentiates your AI application.

There are two approaches to using durable task for AI agents:

- [Durable task extension for Microsoft Agent Framework](./durable-agents-microsoft-agent-framework.md) — Register agents with the extension and they automatically become durable, with persistent sessions, built-in API endpoints, and scaling. No changes to your agent logic are required.
- [Durable Functions and Durable Task SDKs for deterministic agentic workflows](./durable-agents-deterministic-workflows.md) — Build deterministic agentic workflows with any AI framework or direct model API calls using Durable Functions or the Durable Task SDKs.

## What durable execution handles for you

AI agents work well in demos and prototypes, where everything runs in a single process and failures are rare. In production, the picture changes drastically. Durable task on Azure addresses the following challenges so you can focus on your agent's business logic instead of infrastructure plumbing:

### Durable execution — checkpoint and recover automatically

VMs get rebooted for maintenance, processes crash, network connections drop. An agent workflow that ran for two hours consuming thousands of tokens can lose all progress in an instant. Without durable execution, you restart from scratch, re-calling LLMs, re-consuming tokens, and potentially getting different results.

With durable execution, every state change (messages, tool calls, agent decisions) is automatically checkpointed. Workflows survive crashes, reboots, outages, and deployments without losing progress. Completed LLM calls are not repeated, preserving token spend and ensuring consistent results.

The same automatic checkpointing that enables crash recovery also enables long-lived workflows. Agent conversations can span hours, days, or weeks. The full execution state (conversation history, intermediate results, local variables) persists across restarts, redeployments, and infrastructure changes. An agent can pause for human approval, be unloaded from memory entirely, and resume days later with its full context intact. When combined with serverless hosting, you pay nothing for compute while the agent waits.

At the individual task level, durable execution also provides built-in retry policies with configurable backoff strategies. LLM APIs return rate-limit errors, tool calls hit transient network failures, and external services go down temporarily. Durable execution handles these automatically, keeping your agent logic clean.

### Distributed execution — scale automatically

When you move from ten concurrent agent sessions to ten thousand, you need LLM calls, tool invocations, and agent tasks to distribute automatically across available compute. Many AI frameworks leave this as an exercise for the developer.

With durable execution, agent workflows automatically fan out across all available compute instances. LLM calls and tool invocations run on whichever worker has capacity, and if a node goes down mid-inference, the work is reassigned to a healthy one. This enables elastic scaling from a handful of agent sessions to thousands running in parallel, without changes to your application code.

### Determinism — control agent behavior

Non-deterministic LLM responses can lead agents into infinite loops or unexpected decisions. Without explicit control over the execution path, agents become difficult to trust in business-critical workflows.

Durable execution lets you express agentic workflows as deterministic orchestrations written in ordinary imperative code. You define the control flow (which agents to call, in what order, with what inputs) using standard programming constructs like `if/else`, loops, and `try/catch`. This gives you explicit guardrails over agent behavior, ensuring predictable, repeatable execution paths that stakeholders can trust. This complements agent-directed workflows (where the LLM decides what to do next) by providing an outer layer of explicit control when you need it.

### Debuggability — understand what your agents are doing

When an agent behaves unexpectedly, you need to step through what happened: what prompts were sent, what tools were called, what decisions were made, and where things went wrong.

Because orchestrations are written as ordinary code in the language of your choice, you can use familiar development tools (IDEs, debuggers, breakpoints, and unit tests) to develop and troubleshoot agent workflows locally. Set a breakpoint, step through your orchestration, and inspect the state at every decision point just like any other code. The [Durable Task Scheduler dashboard](./durable-task-scheduler/durable-task-scheduler-dashboard.md) extends this into production, providing deep visibility into execution history including inputs, outputs, durations, tool calls, and the full conversation history for each agent session.

## Agentic patterns supported by durable task

Durable task supports patterns that align closely with established agentic workflow designs:

| Agentic pattern | Description |
|---|---|
| **Prompt chaining** | Chain the output of one LLM call as input to the next, with each step checkpointed |
| **Parallelization** | Run multiple LLM calls or agent tasks in parallel across distributed compute, then aggregate results |
| **Routing** | Dynamically select which agent or model to call based on runtime conditions |
| **Orchestrator-worker** | A parent orchestration delegates specialized tasks to child orchestrations or activities |
| **Evaluator-optimizer** | An agent generates content, a second evaluates it, and the loop continues until quality criteria are met |
| **Human-in-the-loop** | Pause execution for human approval or input with configurable timeouts, at zero compute cost |

## Next steps

- [Durable task extension for Microsoft Agent Framework](./durable-agents-microsoft-agent-framework.md) — Durable agents with Microsoft Agent Framework
- [Durable Functions and Durable Task SDKs for deterministic agentic workflows](./durable-agents-deterministic-workflows.md) — Deterministic agentic workflows with any AI framework
- [Durable Task Scheduler overview](./durable-task-scheduler/durable-task-scheduler.md) — Architecture, features, and setup
- [Choose your orchestration framework](./choose-orchestration-framework.md) — Compare Durable Functions and Durable Task SDKs
