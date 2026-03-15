---
title: Durable task for AI agents - Azure
description: Learn how durable execution on Azure provides fault-tolerant, scalable infrastructure for production AI agents using Durable Functions, Durable Task SDKs, and the Durable Task Scheduler.
author: nigreenf
ms.topic: conceptual
ms.date: 03/15/2026
ms.author: nigreenf
---

# Durable task for AI agents

Production AI agents are distributed systems. They call LLMs that can be slow or rate-limited, invoke external tools and APIs that may fail transiently, maintain conversation state across sessions that span hours or weeks, and need to scale across compute instances to handle variable demand. These are the same reliability and coordination challenges that distributed cloud services have faced for years, and they require the same kinds of solutions.

[Durable task on Azure](https://learn.microsoft.com/en-us/azure/azure-functions/durable/) ([Durable Functions](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview), the [Durable Task SDKs](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-task-scheduler/durable-task-scheduler), and the [Durable Task Scheduler](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-task-scheduler/durable-task-scheduler) as the managed backend) provides **durable execution** for Azure: a fault-tolerant approach to running code that automatically handles failures, automatic checkpointing, and distributed coordination. Instead of writing plumbing code for retries, checkpointing state, and error recovery, you offload that complexity to durable task and focus on the business logic that differentiates your AI application.

This article explains how durable execution applies to AI agent scenarios and how to get started with the **durable task extension for Microsoft Agent Framework** or with **Durable Functions and the Durable Task SDKs** for agentic workflows. If you want to jump straight to building:

- [Durable task extension for Microsoft Agent Framework](#durable-task-extension-for-microsoft-agent-framework) for durable agents with Microsoft Agent Framework
- [Durable Functions and Durable Task SDKs for deterministic agentic workflows](#durable-functions-and-durable-task-sdks-for-deterministic-agentic-workflows) with any AI framework or direct model API calls

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

Because orchestrations are written as ordinary code in the language of your choice, you can use familiar development tools (IDEs, debuggers, breakpoints, and unit tests) to develop and troubleshoot agent workflows locally. Set a breakpoint, step through your orchestration, and inspect the state at every decision point just like any other code. The [Durable Task Scheduler dashboard](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-task-scheduler/durable-task-scheduler-dashboard) extends this into production, providing deep visibility into execution history including inputs, outputs, durations, tool calls, and the full conversation history for each agent session.

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

## How to use durable task for AI agents

### Durable task extension for Microsoft Agent Framework

The [durable task extension for Microsoft Agent Framework](https://learn.microsoft.com/en-us/agent-framework/integrations/azure-functions) brings durable execution directly into the [Microsoft Agent Framework](https://learn.microsoft.com/agent-framework/). Register an agent with the extension and it automatically becomes durable, with persistent sessions, built-in API endpoints, and scaling. No changes to your agent logic are required.

The extension supports two hosting models, each available for both .NET and Python:

- **Azure Functions** — Serverless hosting with automatic HTTP endpoints, event-driven scaling, and pay-per-invocation pricing.
- **Any host** — Run durable agents on any compute: Azure Container Apps, Kubernetes, VMs, or locally during development.

#### Durable single agent

Define your agent using the standard Microsoft Agent Framework pattern, then host it with the durable task extension. The extension handles session persistence, endpoint creation, and state management automatically.

**Azure Functions** creates HTTP endpoints and manages serverless scaling automatically:

# [Python](#tab/python)

```python
import os
from agent_framework.azure import AzureOpenAIChatClient, AgentFunctionApp
from azure.identity import DefaultAzureCredential

endpoint = os.getenv("AZURE_OPENAI_ENDPOINT")
deployment_name = os.getenv("AZURE_OPENAI_DEPLOYMENT_NAME", "gpt-4o-mini")

agent = AzureOpenAIChatClient(
    endpoint=endpoint,
    deployment_name=deployment_name,
    credential=DefaultAzureCredential()
).as_agent(
    instructions="You are a professional content writer who creates engaging, "
                 "well-structured documents for any given topic.",
    name="DocumentPublisher"
)

# One line to make the agent durable with serverless hosting
app = AgentFunctionApp(agents=[agent])
```

# [C#](#tab/csharp)

```csharp
var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT") ?? "gpt-4o-mini";

AIAgent agent = new AzureOpenAIClient(new Uri(endpoint), new DefaultAzureCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(
        instructions: "You are a professional content writer who creates engaging, "
                    + "well-structured documents for any given topic.",
        name: "DocumentPublisher");

// One line to make the agent durable with serverless hosting
using IHost app = FunctionsApplication
    .CreateBuilder(args)
    .ConfigureFunctionsWebApplication()
    .ConfigureDurableAgents(options => options.AddAIAgent(agent))
    .Build();
app.Run();
```

---

**Any host** — the same durable agents run on any compute, not just Azure Functions. Deploy to Azure Container Apps, Kubernetes, VMs, or run locally during development:

# [Python](#tab/python)

```python
from agent_framework.azure import AzureOpenAIChatClient, DurableAIAgentWorker
from azure.identity import AzureCliCredential
from durabletask.azuremanaged.worker import DurableTaskSchedulerWorker

agent = AzureOpenAIChatClient(credential=AzureCliCredential()).as_agent(
    name="DocumentPublisher",
    instructions="You are a professional content writer who creates engaging, "
                 "well-structured documents for any given topic.",
)

# Create a worker connected to the Durable Task Scheduler
worker = DurableTaskSchedulerWorker(
    host_address="http://localhost:8080",
    secure_channel=False,
    taskhub="default",
)

# Register the agent and start processing
agent_worker = DurableAIAgentWorker(worker)
agent_worker.add_agent(agent)
worker.start()
```

# [C#](#tab/csharp)

```csharp
var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT") ?? "gpt-4o-mini";

AIAgent agent = new AzureOpenAIClient(new Uri(endpoint), new DefaultAzureCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(
        instructions: "You are a professional content writer who creates engaging, "
                    + "well-structured documents for any given topic.",
        name: "DocumentPublisher");

// Host the agent with Durable Task Scheduler
string connectionString = "Endpoint=http://localhost:8080;TaskHub=default;Authentication=None";

IHost host = Host.CreateDefaultBuilder(args)
    .ConfigureServices(services =>
    {
        services.ConfigureDurableAgents(
            options => options.AddAIAgent(agent),
            workerBuilder: builder => builder.UseDurableTaskScheduler(connectionString),
            clientBuilder: builder => builder.UseDurableTaskScheduler(connectionString));
    })
    .Build();

await host.StartAsync();
```

---

#### Multi-agent orchestration

For deterministic multi-agent workflows, coordinate multiple specialized agents as steps in a durable orchestration. Each agent call is checkpointed, and the orchestration recovers automatically if any step fails. Completed agent calls are not re-executed on recovery.

# [Python](#tab/python)

```python
@app.orchestration_trigger(context_name="context")
def document_publishing_orchestration(context: DurableOrchestrationContext):
    doc_request = context.get_input()

    research_agent = app.get_agent(context, "ResearchAgent")
    writer_agent = app.get_agent(context, "DocumentPublisherAgent")

    # Step 1: Research the topic
    research_result = yield research_agent.run(
        messages=f"Research the following topic: {doc_request.topic}",
        response_schema=ResearchResult
    )

    # Step 2: Generate outline from research
    outline = yield context.call_activity("generate_outline", {
        "topic": doc_request.topic,
        "research_data": research_result.findings
    })

    # Step 3: Write the document
    document = yield writer_agent.run(
        messages=f"""Create a document about {doc_request.topic}.
        Research findings: {research_result.findings}
        Outline: {outline}""",
        response_schema=DocumentResponse
    )

    # Step 4: Publish
    return yield context.call_activity("publish_document", {
        "title": doc_request.topic,
        "content": document.text
    })
```

# [C#](#tab/csharp)

```csharp
[Function(nameof(DocumentPublishingOrchestration))]
public async Task<DocumentResult> DocumentPublishingOrchestration(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var docRequest = context.GetInput<DocumentRequest>();

    var researchAgent = context.GetAgent("ResearchAgent");
    var writerAgent = context.GetAgent("DocumentPublisherAgent");

    // Step 1: Research the topic
    var researchResult = await researchAgent.RunAsync<ResearchResult>(
        $"Research the following topic: {docRequest.Topic}");

    // Step 2: Generate outline from research
    var outline = await context.CallActivityAsync<string>(
        nameof(GenerateOutline),
        new { docRequest.Topic, researchResult.Findings });

    // Step 3: Write the document
    var document = await writerAgent.RunAsync<DocumentResponse>(
        $"""Create a document about {docRequest.Topic}.
        Research findings: {researchResult.Findings}
        Outline: {outline}""");

    // Step 4: Publish
    return await context.CallActivityAsync<DocumentResult>(
        nameof(PublishDocument),
        new { docRequest.Topic, document.Text });
}
```

---

For complete samples across hosting models and languages, see the [.NET Azure Functions](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/AzureFunctions) and [.NET any-host](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/ConsoleApps) samples, or the [Python Azure Functions](https://github.com/microsoft/agent-framework/tree/main/python/samples/04-hosting/azure_functions) and [Python any-host](https://github.com/microsoft/agent-framework/tree/main/python/samples/04-hosting/durabletask) samples on GitHub. To get started, see [Azure Functions (Durable)](https://learn.microsoft.com/en-us/agent-framework/integrations/azure-functions).

#### Durable Task Scheduler dashboard

The [Durable Task Scheduler dashboard](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-task-scheduler/durable-task-scheduler-dashboard) gives you full visibility into your durable agents: view conversation history for each agent session, inspect tool calls and structured outputs, trace multi-agent orchestration flows, and monitor performance metrics. Both local development (via the emulator) and production deployments surface the same dashboard experience.

:::image type="content" source="media/durable-tasks-for-ai-agents/dashboard-agent.png" alt-text="Screenshot of the Durable Task Scheduler dashboard showing agent conversation history and session details." lightbox="media/durable-tasks-for-ai-agents/dashboard-agent.png":::

:::image type="content" source="media/durable-tasks-for-ai-agents/dashboard-orchestration.png" alt-text="Screenshot of the Durable Task Scheduler dashboard showing a deterministic agentic orchestration view." lightbox="media/durable-tasks-for-ai-agents/dashboard-orchestration.png":::

### Durable Functions and Durable Task SDKs for deterministic agentic workflows

You don't need to use the Microsoft Agent Framework to benefit from durable execution. Using [Durable Functions](https://learn.microsoft.com/en-us/azure/azure-functions/durable/) or the [Durable Task SDKs](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-task-scheduler/durable-task-scheduler), you can build deterministic agentic workflows with any AI framework or direct model API calls. This approach gives you full control over orchestration logic and supports .NET, Python, Java, TypeScript/JavaScript, Go, and PowerShell.

The following example shows an orchestration that chains LLM-powered agent activities and includes human-in-the-loop approval. Each activity function wraps an AI agent call (using any framework or direct model API), and the orchestration controls the overall flow:

# [Python](#tab/python)

```python
# Activity function: wraps an AI agent call as a durable activity
@app.activity_trigger(input_name="topic")
def generate_content_agent(topic: str) -> str:
    # Call your agent here using any framework (Semantic Kernel, LangChain, etc.)
    # or call the LLM directly via the Azure OpenAI SDK
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": f"Write an article about {topic}"}]
    )
    return response.choices[0].message.content

# Orchestration: controls the overall workflow
@app.orchestration_trigger(context_name="context")
def content_review_orchestration(context: DurableOrchestrationContext):
    topic = context.get_input()

    # Step 1: Call an AI agent to generate content
    draft = yield context.call_activity("generate_content_agent", topic)

    # Step 2: Wait for human approval (zero compute cost while waiting)
    yield context.call_activity("notify_reviewer", draft)
    approval = yield context.wait_for_external_event("ApprovalEvent")

    # Step 3: Call an AI agent to refine, or reject
    if approval["approved"]:
        return yield context.call_activity("publish_content_agent", draft)

    return {"status": "rejected", "feedback": approval["feedback"]}
```

# [C#](#tab/csharp)

```csharp
// Activity function: wraps an AI agent call as a durable activity
[Function(nameof(GenerateContentAgent))]
public async Task<string> GenerateContentAgent(
    [ActivityTrigger] string topic)
{
    // Call your agent here using any framework (Semantic Kernel, AutoGen, etc.)
    // or call the LLM directly via the Azure OpenAI SDK
    var response = await chatClient.CompleteChatAsync(
        [new UserChatMessage($"Write an article about {topic}")]);
    return response.Value.Content[0].Text;
}

// Orchestration: controls the overall workflow
[Function(nameof(ContentReviewOrchestration))]
public async Task<string> ContentReviewOrchestration(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    string topic = context.GetInput<string>();

    // Step 1: Call an AI agent to generate content
    string draft = await context.CallActivityAsync<string>(
        nameof(GenerateContentAgent), topic);

    // Step 2: Wait for human approval (zero compute cost while waiting)
    await context.CallActivityAsync(nameof(NotifyReviewer), draft);
    var approval = await context.WaitForExternalEvent<ApprovalResponse>("ApprovalEvent");

    // Step 3: Call an AI agent to refine, or reject
    if (approval.Approved)
        return await context.CallActivityAsync<string>(nameof(PublishContentAgent), draft);

    return $"Rejected: {approval.Feedback}";
}
```

---

## Get started

- [Azure Functions (Durable)](https://learn.microsoft.com/en-us/agent-framework/integrations/azure-functions) — Tutorials, code samples, and hosting guide for durable agents with Microsoft Agent Framework
- [.NET samples: Azure Functions](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/AzureFunctions) | [Any host](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/ConsoleApps) — .NET durable agent samples across hosting models
- [Python samples: Azure Functions](https://github.com/microsoft/agent-framework/tree/main/python/samples/04-hosting/azure_functions) | [Any host](https://github.com/microsoft/agent-framework/tree/main/python/samples/04-hosting/durabletask) — Python durable agent samples across hosting models
- [Durable Task Scheduler overview](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-task-scheduler/durable-task-scheduler) — Architecture, features, and setup
- [Choose your orchestration framework](https://learn.microsoft.com/en-us/azure/azure-functions/durable/choose-orchestration-framework) — Compare Durable Functions and Durable Task SDKs
