---
title: Durable Task for AI agents - Azure
description: Learn how durable execution on Azure provides fault-tolerant, scalable infrastructure for production AI agents using Durable Functions, Durable Task SDKs, and the Durable Task Scheduler.
author: greenie-msft
ms.topic: conceptual
ms.date: 03/15/2026
ms.author: nigreenf
---

# Durable Task for AI agents

Production AI agents often face the same reliability and coordination problems faced by distributed cloud services, including:
- Calling large language models (LLMs) that can be slow or rate-limited.
- Invoking external tools and APIs that might fail transiently.
- Maintaining conversation state across sessions that span hours or weeks.
- Needing to scale across compute instances to handle variable demand.

These challenges can be solved by **durable execution**, a fault-tolerant approach to running code that automatically handles failures, checkpointing, and distributed coordination. 

In this article, you'll walk through:

> [!div class="checklist"]
>
> - Achieving durable execution using Durable Task.
> - Production challenges durable execution solves.
> - The workflow approaches and agentic patterns Durable Task supports.
> - Available tools for debugging and observability.

## Durable Task

You can achieve durable execution in your scenario using [Durable Task](../sdks/durable-task-overview.md). Offload plumbing code for retries, state checkpointing, and error recovery to Durable Task and focus on the business logic that differentiates your AI application.

Durable Task includes two hosting options:
- [Durable Functions](../../azure-functions/durable-functions/durable-functions-overview.md) (an extension of Azure Functions) 
- The [Durable Task SDKs](../sdks/durable-task-overview.md) (supporting Azure Container Apps, Azure App Service, Azure Kubernetes Service, and other computes)

Durable Task is backed by the [Durable Task Scheduler](../scheduler/durable-task-scheduler.md) as the managed storage provider.

## Production challenges durable execution solves

AI agents work well locally, but experience challenges once in production. Durable Task on Azure addresses the following challenges so you can focus on your agent's business logic instead of infrastructure plumbing.

### Automatic checkpointing and recovery

**Scenario:**  
The virtual machines or containers running your agent code reboot for maintenance, crash unexpectedly, or lose network connectivity. An agent workflow that ran for two hours and consumed thousands of LLM tokens loses all progress in an instant. 

**Without durable execution:**  
You must restart from scratch: re-calling LLMs, re-consuming tokens, and potentially getting different results.

**With durable execution:**  
Every state change in your agent's orchestration (messages, tool call results, and agent decisions) is automatically checkpointed. When the compute running your orchestration restarts, execution resumes from the last checkpoint. Completed LLM calls aren't repeated, which preserves token spend and ensures consistent results.

This checkpointing also enables long-lived workflows. Agent conversations can span hours, days, or weeks. The full execution state (including conversation history, intermediate results, and local variables), persists across restarts, redeployments, and infrastructure changes. An agent can pause for human approval, be unloaded from memory entirely, and resume days later with its full context intact. When you combine this capability with serverless hosting, you pay nothing for compute while the agent waits.

At the individual task level, durable execution provides built-in retry policies with configurable backoff strategies. LLM APIs return rate-limit errors, tool calls encounter transient network failures, and external services go down temporarily. Durable execution handles these situations automatically, which keeps your agent logic clean.

### Distributed scaling

**Scenario:**  
You move from 10 concurrent agent sessions to 10,000. All LLM calls, tool invocations, and agent tasks need to distribute automatically across available computes. 

**Without durable execution:**  
Usually, you're expected to managed this distribution yourself. 

**With durable execution:**  
Agent workflows automatically fan out across all available compute instances. LLM calls and tool invocations run on whichever worker has capacity. If a node goes down mid-execution, the work is reassigned to a healthy one. This approach enables elastic scaling from a handful of agent sessions to thousands running in parallel, without changes to your application code.

### Deterministic control flow

**Scenario:**
A non-deterministic LLM responses led your agents into an infinite loops or an unexpected decisios. 

**Without durable execution:**
Without explicit control over the execution path, agents become difficult to trust in business-critical workflows.

**With durable execution:**  
You can express agentic workflows as deterministic orchestrations written in ordinary imperative code. You define the control flow, including which agents to call, in what order, and with what inputs, by using standard programming constructs like `if/else`, loops, and `try/catch`. This approach gives you explicit guardrails over agent behavior, which ensures predictable, repeatable execution paths that stakeholders can trust. Deterministic orchestrations complement agent-directed workflows (where the LLM decides what to do next) by providing an outer layer of explicit control when you need it.

### Debugging and observability

**Scenario:**
An agent is behaving unexpectedly

**Without durable execution:**
You step through what happened locally. You manually check: 
- What prompts were sent
- What tools were called
- What decisions were made
- Where things went wrong

**With durable execution:**
Orchestrations are written as ordinary code in the language of your choice. You can use familiar development tools like IDEs, debuggers, breakpoints, and unit tests to develop and troubleshoot agent workflows locally:
1. Set a breakpoint.
1. Step through your orchestration.
1. Inspect the state at every decision point, just like any other code. 

The [Durable Task Scheduler dashboard](../scheduler/durable-task-scheduler-dashboard.md) extends this capability into production. It provides deep visibility into execution history, including inputs, outputs, durations, tool calls, and the full conversation history for each agent session.

## Supported agentic workflow approaches

With an understanding of these production challenges, you can choose how to build your agentic workflows. Durable Task supports the two general approaches to building agentic systems:

| Approach | Descriptions | Learn how with Durable Task... |
| -------- | ------------ | ------------------------------ |
| **Agent-directed workflows** | The LLM drives the control flow. The agent decides which tools to call, in what order, and when the task is complete. The developer provides tools and instructions, but the agent determines the execution path at runtime. | [Durable task extension for Microsoft Agent Framework (Preview)](./durable-agents-microsoft-agent-framework.md) |
| **Deterministic workflows** | Your code defines the control flow. You write the exact sequence of steps, including branching, parallelism, and error handling, using standard programming constructs like `if/else`, loops, and `try/catch`. The LLM is called as a step within the workflow, but doesn't control the overall flow. | [Durable Functions and Durable Task SDKs for deterministic agentic workflows](./durable-agents-deterministic-workflows.md) |

Both agentic workflow approaches provide durable execution and can be used together in the same application. 

### Compare agentic workflow options on Azure

With these two options, you can choose between several models for building agentic workflows on Azure, including:
- [The Durable Task programming model](../common/programming-model-overview.md)
- [Microsoft Agent Framework (MAF) workflows](/agent-framework/workflows/)
- [Logic Apps _agent loop_](../../logic-apps/agent-workflows-concepts.md)

Compare these options in the following table to decide which one best fits your needs.

| Capability | Durable Task | Microsoft Agent Framework workflows | Logic Apps Agent Loop |
| --- | --- | --- | --- |
| **Control flow** | Code-defined (imperative) | Code-defined (graphs) | Designer / declarative (JSON) |
| **Programming languages** | .NET, Python, Java, TypeScript/JavaScript, PowerShell | .NET, Python | Visual designer / JSON |
| **AI framework support** | Any framework (Semantic Kernel, LangChain, AutoGen, etc.) or direct model API calls | Optimized for Microsoft Agent Framework | Built-in AI connectors |
| **Hosting** | Azure Functions (via Durable Functions) or any host (via Durable Task SDKs) | Any, with first-class [Foundry Hosted Agents](/azure/foundry/agents/concepts/hosted-agents) support | Azure Logic Apps managed service (Consumption or Standard SKU) |
| **State storage** | Durable Task Scheduler (managed) | Bring your own (extensible via checkpoint manager) | Logic Apps runtime (managed) |
| **Agent-directed workflows** | Not built-in (deterministic only) | Yes, via the [Durable Task extension for MAF](./durable-agents-microsoft-agent-framework.md) | Yes, via the Agent Loop action |
| **Target audience** | Backend developers | Application developers | Integration developers / low-code users |
| **Long-running tasks** | First-class (hours / days / weeks / eternal) | Supported via developer-controlled workflow state checkpointing | Supported for *stateful* workflows only (up to 90 days) |
| **Recovery from failure** | Automatic | Manual | Automatic |
| **Observability** | Execution history in the Durable Task Scheduler dashboard, OpenTelemetry | OpenTelemetry, custom visualization | Azure Monitor / Logic Apps diagnostics |
| **Agent loops** | Build your own using orchestrations and entities | Built-in. Agents run in a loop where the LLM decides what to do next, calling tools, responding to users, and continuing until the task is complete | Build workflows that can use agent loops and integrate across services, systems, apps, and data |

## Supported agentic patterns

Durable Task supports patterns that align closely with established agentic workflow designs. 

| Agentic pattern | Description | Example use cases |
| --- | --- | --- |
| **Prompt chaining** | Chain the output of one LLM call as input to the next, with each step checkpointed. | Multi-step document processing, sequential data enrichment |
| **Parallelization** | Run multiple LLM calls or agent tasks in parallel across distributed compute, then aggregate results. | Batch analysis of documents, parallel tool calls, multi-model voting |
| **Routing** | Dynamically select which agent or model to call based on runtime conditions. | Intent classification to specialized agents, model selection based on task complexity |
| **Orchestrator-worker** | A parent orchestration delegates specialized tasks to child orchestrations or activities. | Research pipelines, multi-agent collaboration |
| **Evaluator-optimizer** | One agent generates content, a second evaluates it, and the loop continues until quality criteria are met. | Code generation with review, iterative content refinement |
| **Human-in-the-loop** | Pause execution for human approval or input with configurable timeouts, at zero compute cost. | Expense approvals, content moderation, escalation workflows |

See Anthropic's [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) for an overview of these patterns.

## Next steps

- [Durable Functions and Durable Task SDKs for deterministic agentic workflows](./durable-agents-deterministic-workflows.md)
- [Durable task extension for Microsoft Agent Framework (Preview)](./durable-agents-microsoft-agent-framework.md)
- [Durable Task Scheduler overview](../scheduler/durable-task-scheduler.md)