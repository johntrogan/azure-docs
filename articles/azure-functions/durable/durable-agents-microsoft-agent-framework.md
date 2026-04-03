---
title: Durable task extension for Microsoft Agent Framework - Azure
description: Learn how to use the durable task extension for Microsoft Agent Framework to build fault-tolerant, scalable AI agents with persistent sessions and automatic checkpointing.
author: greenie-msft
ms.topic: conceptual
ms.date: 03/15/2026
ms.author: nigreenf
---

# Durable task extension for Microsoft Agent Framework (Preview)

The [durable task extension for Microsoft Agent Framework](/agent-framework/integrations/azure-functions) brings [durable execution](./durable-task-for-ai-agents.md) directly into the [Microsoft Agent Framework](/agent-framework/). Register an agent with the extension and it automatically becomes durable, with persistent sessions, built-in API endpoints, and scaling. No changes to your agent logic are required.

The extension supports two hosting models:

- **Azure Functions** — Serverless hosting with automatic HTTP endpoints, event-driven scaling, and pay-per-invocation pricing.
- **Bring your own compute** — Run durable agents on any compute: Azure Container Apps, Azure App Service, Kubernetes, bare-metal servers, or locally on a laptop (typically during development).

## Durable single agent

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

## Multi-agent orchestration

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

For complete samples across hosting models and languages, see the [.NET Azure Functions](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/AzureFunctions) and [.NET any-host](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/ConsoleApps) samples, or the [Python Azure Functions](https://github.com/microsoft/agent-framework/tree/main/python/samples/04-hosting/azure_functions) and [Python any-host](https://github.com/microsoft/agent-framework/tree/main/python/samples/04-hosting/durabletask) samples on GitHub. To get started, see [Azure Functions (Durable)](/agent-framework/integrations/azure-functions).

## Durable Task Scheduler dashboard

The [Durable Task Scheduler dashboard](./durable-task-scheduler/durable-task-scheduler-dashboard.md) gives you full visibility into your durable agents: view conversation history for each agent session, inspect tool calls and structured outputs, trace multi-agent orchestration flows, and monitor performance metrics. Both local development (via the emulator) and production deployments surface the same dashboard experience.

:::image type="content" source="media/durable-task-for-ai-agents/dashboard-agent.png" alt-text="Screenshot of the Durable Task Scheduler dashboard showing agent conversation history and session details." lightbox="media/durable-task-for-ai-agents/dashboard-agent.png":::

:::image type="content" source="media/durable-task-for-ai-agents/dashboard-orchestration.png" alt-text="Screenshot of the Durable Task Scheduler dashboard showing a deterministic agentic orchestration view." lightbox="media/durable-task-for-ai-agents/dashboard-orchestration.png":::

## Session time-to-live (TTL)

Durable agent sessions automatically maintain conversation history and state. Without automatic cleanup, this state can accumulate indefinitely, consuming storage resources and increasing costs. The time-to-live (TTL) feature provides automatic cleanup of idle agent sessions.

When an agent session is idle (no messages sent to it) for longer than the configured TTL period, the session state is automatically deleted. Each new interaction resets the TTL timer, extending the session's lifetime.

### Default values

- **Default TTL**: 14 days
- **Minimum TTL deletion delay**: 5 minutes

### Configuration

TTL can be configured globally or per-agent. When an agent session expires, its entire state is deleted, including conversation history and any custom state data. If a message is sent to the same session after deletion, a new session is created with a fresh conversation history.

> [!NOTE]
> TTL configuration is currently available in .NET only. Python support is planned for a future release.

```csharp
services.ConfigureDurableAgents(
    options =>
    {
        // Set global default TTL to 7 days
        options.DefaultTimeToLive = TimeSpan.FromDays(7);

        // Agent with custom TTL of 1 day
        options.AddAIAgent(shortLivedAgent, timeToLive: TimeSpan.FromDays(1));

        // Agent with custom TTL of 90 days
        options.AddAIAgent(longLivedAgent, timeToLive: TimeSpan.FromDays(90));

        // Agent using global default (7 days)
        options.AddAIAgent(defaultAgent);

        // Agent with no TTL (never expires)
        options.AddAIAgent(permanentAgent, timeToLive: null);
    });
```

## Known limitations

**Maximum conversation size.** Agent session state, including the full conversation history, is subject to the state-size limits of the durable backend. When using the [Durable Task Scheduler](./durable-task-scheduler/durable-task-scheduler-overview.md), the maximum entity state size is 1 MB. Long-running conversations with large tool call responses may reach this limit. Compaction of conversation history must be done manually, for example, by starting a new agent session and summarizing the prior context.

**Latency.** All agent interactions are routed through the Durable Task Scheduler, which adds latency compared to in-memory agent execution. This tradeoff provides durability and distributed scaling.

**Streaming.** Because durable agents are implemented on top of durable entities, the underlying communication model is request/response. Streaming is supported through response callbacks (for example, pushing tokens to a Redis Stream for client consumption), while the entity returns the complete response after the stream finishes.

**TTL expiration.** The TTL timer is based on wall-clock time since the last message, not cumulative activity time. Once a session is deleted (via TTL expiration or manual deletion), its conversation history can't be recovered.

## Next steps

- [Durable task for AI agents overview](./durable-task-for-ai-agents.md) — What durable execution handles and agentic patterns
- [Durable Functions and Durable Task SDKs for deterministic agentic workflows](./durable-agents-deterministic-workflows.md) — Build agentic workflows with any AI framework
- [Azure Functions (Durable)](/agent-framework/integrations/azure-functions) — Tutorials, code samples, and hosting guide
- [Durable Task Scheduler overview](./durable-task-scheduler/durable-task-scheduler.md) — Architecture, features, and setup
