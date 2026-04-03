---
title: Durable Task for AI agents - Azure
description: Learn how durable execution on Azure provides fault-tolerant, scalable infrastructure for production AI agents using Durable Functions, Durable Task SDKs, and the Durable Task Scheduler.
author: greenie-msft
ms.topic: conceptual
ms.date: 03/15/2026
ms.author: nigreenf
---

# Durable Task for AI agents

Production AI agents are often distributed systems. They call large language models (LLMs) that can be slow or rate-limited, invoke external tools and APIs that might fail transiently, maintain conversation state across sessions that span hours or weeks, and need to scale across compute instances to handle variable demand. These challenges mirror the reliability and coordination problems that distributed cloud services have faced for years, and they require the same kinds of solutions.

[Durable Task on Azure](./index.yml) ([Durable Functions](./durable-functions-overview.md), the [Durable Task SDKs](./durable-task-scheduler/durable-task-overview.md), and the [Durable Task Scheduler](./durable-task-scheduler/durable-task-scheduler.md) as the managed backend) provides **durable execution** for Azure. Durable execution is a fault-tolerant approach to running code that automatically handles failures, checkpointing, and distributed coordination. Instead of writing plumbing code for retries, state checkpointing, and error recovery, you offload that complexity to Durable Task and focus on the business logic that differentiates your AI application.

There are two general approaches to building agentic systems:

- **Agent-directed workflows**: The LLM drives the control flow. The agent decides which tools to call, in what order, and when the task is complete. The developer provides tools and instructions, but the agent determines the execution path at runtime.
- **Deterministic workflows**: Your code defines the control flow. You write the exact sequence of steps, including branching, parallelism, and error handling, using standard programming constructs like `if/else`, loops, and `try/catch`. The LLM is called as a step within the workflow, but doesn't control the overall flow.

Durable Task on Azure supports both approaches:

- [Durable Functions and Durable Task SDKs for deterministic agentic workflows](./durable-agents-deterministic-workflows.md) — Build deterministic agentic workflows with any AI framework using Durable Functions or the Durable Task SDKs, with full control over orchestration logic.
- [Durable Task extension for Microsoft Agent Framework (Preview)](./durable-agents-microsoft-agent-framework.md) — Register agents with the extension and they automatically become durable, with persistent sessions, built-in API endpoints, and scaling. No changes to your agent logic are required.

## How durable execution works

AI agents work well in demos and prototypes, where everything runs in a single process and failures are rare. In production, the picture changes. Durable Task on Azure addresses the following challenges so you can focus on your agent's business logic instead of infrastructure plumbing.

### Automatic checkpointing and recovery

The VMs or containers running your agent code can be rebooted for maintenance, crash unexpectedly, or lose network connectivity. An agent workflow that ran for two hours and consumed thousands of LLM tokens can lose all progress in an instant. Without durable execution, you must restart from scratch, re-calling LLMs, re-consuming tokens, and potentially getting different results.

With durable execution, every state change in your agent's orchestration (messages, tool call results, and agent decisions) is automatically checkpointed. When the compute running your orchestration restarts, execution resumes from the last checkpoint. Completed LLM calls aren't repeated, which preserves token spend and ensures consistent results.

This checkpointing also enables long-lived workflows. Agent conversations can span hours, days, or weeks. The full execution state, including conversation history, intermediate results, and local variables, persists across restarts, redeployments, and infrastructure changes. An agent can pause for human approval, be unloaded from memory entirely, and resume days later with its full context intact. When you combine this capability with serverless hosting, you pay nothing for compute while the agent waits.

At the individual task level, durable execution provides built-in retry policies with configurable backoff strategies. LLM APIs return rate-limit errors, tool calls encounter transient network failures, and external services go down temporarily. Durable execution handles these situations automatically, which keeps your agent logic clean.

### Distributed scaling

When you move from 10 concurrent agent sessions to 10,000, LLM calls, tool invocations, and agent tasks need to distribute automatically across available compute. Many AI frameworks leave this distribution as an exercise for the developer.

With durable execution, agent workflows automatically fan out across all available compute instances. LLM calls and tool invocations run on whichever worker has capacity. If a node goes down mid-execution, the work is reassigned to a healthy one. This approach enables elastic scaling from a handful of agent sessions to thousands running in parallel, without changes to your application code.

### Deterministic control flow

Non-deterministic LLM responses can lead agents into infinite loops or unexpected decisions. Without explicit control over the execution path, agents become difficult to trust in business-critical workflows.

Durable execution lets you express agentic workflows as deterministic orchestrations written in ordinary imperative code. You define the control flow, including which agents to call, in what order, and with what inputs, by using standard programming constructs like `if/else`, loops, and `try/catch`. This approach gives you explicit guardrails over agent behavior, which ensures predictable, repeatable execution paths that stakeholders can trust. Deterministic orchestrations complement agent-directed workflows (where the LLM decides what to do next) by providing an outer layer of explicit control when you need it.

### Debugging and observability

When an agent behaves unexpectedly, you need to step through what happened: what prompts were sent, what tools were called, what decisions were made, and where things went wrong.

Because orchestrations are written as ordinary code in the language of your choice, you can use familiar development tools like IDEs, debuggers, breakpoints, and unit tests to develop and troubleshoot agent workflows locally. Set a breakpoint, step through your orchestration, and inspect the state at every decision point just like any other code. The [Durable Task Scheduler dashboard](./durable-task-scheduler/durable-task-scheduler-dashboard.md) extends this capability into production and provides deep visibility into execution history, including inputs, outputs, durations, tool calls, and the full conversation history for each agent session.

## Agentic patterns supported by Durable Task

Durable Task supports patterns that align closely with established agentic workflow designs (see Anthropic's [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) for an overview of these patterns):

| Agentic pattern | Description | Example use cases |
|---|---|---|
| **Prompt chaining** | Chain the output of one LLM call as input to the next, with each step checkpointed. | Multi-step document processing, sequential data enrichment |
| **Parallelization** | Run multiple LLM calls or agent tasks in parallel across distributed compute, then aggregate results. | Batch analysis of documents, parallel tool calls, multi-model voting |
| **Routing** | Dynamically select which agent or model to call based on runtime conditions. | Intent classification to specialized agents, model selection based on task complexity |
| **Orchestrator-worker** | A parent orchestration delegates specialized tasks to child orchestrations or activities. | Research pipelines, multi-agent collaboration |
| **Evaluator-optimizer** | One agent generates content, a second evaluates it, and the loop continues until quality criteria are met. | Code generation with review, iterative content refinement |
| **Human-in-the-loop** | Pause execution for human approval or input with configurable timeouts, at zero compute cost. | Expense approvals, content moderation, escalation workflows |

## Compare approaches

Both approaches provide durable execution (automatic checkpointing, crash recovery, distributed scaling) and can be used together in the same application. The key differences are:

| | **Durable Functions / Durable Task SDKs** | **Microsoft Agent Framework extension (Preview)** |
|---|---|---|
| **Agent loops** | Build your own using orchestrations and entities | Built-in. Agents run in a loop where the LLM decides what to do next, calling tools, responding to users, and continuing until the task is complete. |
| **AI framework** | Any framework (Semantic Kernel, LangChain, AutoGen, etc.) or direct model API calls | Microsoft Agent Framework (required) |
| **Language support** | .NET, Python, Java, TypeScript/JavaScript, Go, PowerShell | .NET, Python |
| **Hosting** | Azure Functions or any host (via Durable Task SDKs) | Azure Functions or any host |

## Related content

- [Durable Functions and Durable Task SDKs for deterministic agentic workflows](./durable-agents-deterministic-workflows.md) — Build deterministic agentic workflows with full control over orchestration logic
- [Durable Task extension for Microsoft Agent Framework (Preview)](./durable-agents-microsoft-agent-framework.md) — Make agents durable with persistent sessions and built-in API endpoints
- [Durable Task Scheduler overview](./durable-task-scheduler/durable-task-scheduler.md) — Architecture, features, and setup
