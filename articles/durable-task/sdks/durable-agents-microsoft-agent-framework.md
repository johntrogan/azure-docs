---
title: Durable Task extension for Microsoft Agent Framework - Azure
description: Learn how to use the Durable Task extension for Microsoft Agent Framework to build fault-tolerant, scalable AI agents with persistent sessions and automatic checkpointing.
author: greenie-msft
ms.topic: conceptual
ms.date: 04/07/2026
ms.author: nigreenf
zone_pivot_groups: agent-framework-approach
---

# Durable Task extension for Microsoft Agent Framework (Preview)

The [Durable Task extension for Microsoft Agent Framework](/agent-framework/integrations/azure-functions) brings [durable execution](./durable-task-for-ai-agents.md) directly into the [Microsoft Agent Framework](/agent-framework/). You can register agents with the extension to make them automatically durable with persistent sessions, built-in API endpoints, and distributed scaling — without changes to your agent logic.

The extension internally implements [entity-based agent loops](./durable-agents-patterns.md#entity-based-agent-loops), where each agent session is a durable entity that automatically manages conversation state and checkpointing.

The extension supports two hosting approaches:

- **Azure Functions** using the Azure Functions integration package.
- **Bring your own compute** using the base package.

## Agent hosting

Define your agent using the standard Microsoft Agent Framework pattern, then enhance it with the Durable Task extension. The extension handles session persistence, endpoint creation, and state management automatically.

::: zone pivot="azure-functions"

# [C#](#tab/csharp)

```csharp
var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("AZURE_OPENAI_ENDPOINT is not set.");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME")
    ?? throw new InvalidOperationException("AZURE_OPENAI_DEPLOYMENT_NAME is not set.");

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

# [Python](#tab/python)

```python
import os
from agent_framework.azure import AzureOpenAIChatClient, AgentFunctionApp
from azure.identity import DefaultAzureCredential

endpoint = os.getenv("AZURE_OPENAI_ENDPOINT")
deployment_name = os.getenv("AZURE_OPENAI_DEPLOYMENT_NAME")

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

---

::: zone-end

::: zone pivot="other-compute"

# [C#](#tab/csharp)

```csharp
var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("AZURE_OPENAI_ENDPOINT is not set.");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME")
    ?? throw new InvalidOperationException("AZURE_OPENAI_DEPLOYMENT_NAME is not set.");

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

---

::: zone-end

## Multi-agent orchestration

You can coordinate multiple specialized agents as steps in a durable orchestration. Each agent call is checkpointed, and the orchestration recovers automatically if any step fails. Completed agent calls aren't re-executed on recovery.

The following example shows a sequential multi-agent workflow where a research agent gathers information and a writer agent produces a document.

::: zone pivot="azure-functions"

# [C#](#tab/csharp)

```csharp
[Function(nameof(DocumentPublishingOrchestration))]
public async Task<string> DocumentPublishingOrchestration(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var docRequest = context.GetInput<DocumentRequest>();

    DurableAIAgent researchAgent = context.GetAgent("ResearchAgent");
    DurableAIAgent writerAgent = context.GetAgent("DocumentPublisherAgent");

    // Step 1: Research the topic
    AgentResponse<ResearchResult> researchResult = await researchAgent
        .RunAsync<ResearchResult>(
            $"Research the following topic: {docRequest.Topic}");

    // Step 2: Write the document using the research findings
    AgentResponse<DocumentResponse> document = await writerAgent
        .RunAsync<DocumentResponse>(
            $"""Create a document about {docRequest.Topic}.
            Research findings: {researchResult.Result.Findings}""");

    // Step 3: Publish
    return await context.CallActivityAsync<string>(
        nameof(PublishDocument),
        new { docRequest.Topic, document.Result.Text });
}
```

# [Python](#tab/python)

```python
@app.orchestration_trigger(context_name="context")
def document_publishing_orchestration(context: DurableOrchestrationContext):
    doc_request = context.get_input()

    research_agent = app.get_agent(context, "ResearchAgent")
    writer_agent = app.get_agent(context, "DocumentPublisherAgent")

    research_session = research_agent.create_session()
    writer_session = writer_agent.create_session()

    # Step 1: Research the topic
    research_result = yield research_agent.run(
        messages=f"Research the following topic: {doc_request['topic']}",
        session=research_session,
    )

    # Step 2: Write the document using the research findings
    document = yield writer_agent.run(
        messages=f"""Create a document about {doc_request['topic']}.
        Research findings: {research_result.text}""",
        session=writer_session,
    )

    # Step 3: Publish
    return (yield context.call_activity("publish_document", {
        "title": doc_request["topic"],
        "content": document.text
    }))
```

---

::: zone-end

::: zone pivot="other-compute"

# [C#](#tab/csharp)

```csharp
static async Task<string> DocumentPublishingOrchestration(
    TaskOrchestrationContext context, DocumentRequest docRequest)
{
    DurableAIAgent researchAgent = context.GetAgent("ResearchAgent");
    DurableAIAgent writerAgent = context.GetAgent("DocumentPublisherAgent");

    // Step 1: Research the topic
    AgentResponse<ResearchResult> researchResult = await researchAgent
        .RunAsync<ResearchResult>(
            $"Research the following topic: {docRequest.Topic}");

    // Step 2: Write the document using the research findings
    AgentResponse<DocumentResponse> document = await writerAgent
        .RunAsync<DocumentResponse>(
            $"""Create a document about {docRequest.Topic}.
            Research findings: {researchResult.Result.Findings}""");

    // Step 3: Publish
    return await context.CallActivityAsync<string>(
        nameof(PublishDocument),
        new { docRequest.Topic, document.Result.Text });
}
```

# [Python](#tab/python)

```python
from agent_framework.azure import DurableAIAgentOrchestrationContext

def document_publishing_orchestration(ctx, doc_request: dict):
    agent_context = DurableAIAgentOrchestrationContext(ctx)

    research = agent_context.get_agent("ResearchAgent")
    writer = agent_context.get_agent("DocumentPublisherAgent")

    research_session = research.create_session()
    writer_session = writer.create_session()

    # Step 1: Research the topic
    research_result = yield research.run(
        messages=f"Research the following topic: {doc_request['topic']}",
        session=research_session,
    )

    # Step 2: Write the document using the research findings
    document = yield writer.run(
        messages=f"""Create a document about {doc_request['topic']}.
        Research findings: {research_result.text}""",
        session=writer_session,
    )

    # Step 3: Publish
    return (yield ctx.call_activity(publish_document, input={
        "title": doc_request["topic"],
        "content": document.text
    }))
```

---

::: zone-end

## Durable Task Scheduler dashboard

Use the [Durable Task Scheduler dashboard](../scheduler/durable-task-scheduler-dashboard.md) for full visibility into your durable agents: 
- View conversation history for each agent session
- Inspect tool calls and structured outputs
- Trace multi-agent orchestration flows
- Monitor performance metrics

Both local development (via the emulator) and production deployments surface the same dashboard experience.

:::image type="content" source="media/durable-task-for-ai-agents/dashboard-agent.png" alt-text="Screenshot of the Durable Task Scheduler dashboard showing agent conversation history and session details." lightbox="media/durable-task-for-ai-agents/dashboard-agent.png":::

:::image type="content" source="media/durable-task-for-ai-agents/dashboard-orchestration.png" alt-text="Screenshot of the Durable Task Scheduler dashboard showing a deterministic agentic orchestration view." lightbox="media/durable-task-for-ai-agents/dashboard-orchestration.png":::

## Session time-to-live (TTL)

Durable agent sessions automatically maintain conversation history and state, which can accumulate indefinitely. The time-to-live (TTL) feature provides automatic cleanup of idle sessions, preventing storage resource consumption and increased costs. 

When an agent session is idle for longer than the configured TTL period, the session state is automatically deleted. Each new interaction resets the TTL timer, extending the session's lifetime.

### Default values

- **Default TTL**: 14 days
- **Minimum TTL deletion delay**: 5 minutes

### Configuration

TTL can be configured globally or per-agent. When an agent session expires, its entire state is deleted, including conversation history and any custom state data. If a message is sent to the same session after deletion, a new session is created with a fresh conversation history.

> [!NOTE]
> TTL configuration is currently available in .NET only.

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

- **Maximum conversation size.**  
    Agent session state, including the full conversation history, is subject to the state-size limits of the durable backend. When using the [Durable Task Scheduler](../scheduler/durable-task-scheduler.md), the maximum entity state size is 1 MB. Long-running conversations with large tool call responses may reach this limit. Compaction of conversation history must be done manually, for example, by starting a new agent session and summarizing the prior context.

- **Latency.**  
   All agent interactions are routed through the Durable Task Scheduler, which adds latency compared to in-memory agent execution. This tradeoff provides durability and distributed scaling.

- **Streaming.**  
   Since durable agents are implemented on top of durable entities, the underlying communication model is request/response. Streaming is supported through response callbacks (for example, pushing tokens to a Redis Stream for client consumption), while the entity returns the complete response after the stream finishes.

- **TTL expiration.**  
   The TTL timer is based on wall-clock time since the last message, not cumulative activity time. Once a session is deleted (via TTL expiration or manual deletion), its conversation history can't be recovered.

## Related links

::: zone pivot="azure-functions"

For complete code samples:
- [.NET Azure Functions](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/AzureFunctions)
- [Python Azure Functions](https://github.com/microsoft/agent-framework/tree/main/python/samples/04-hosting/azure_functions) 

::: zone-end

::: zone pivot="other-compute"

For complete code samples:
- [.NET any-host](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/ConsoleApps)
- [Python any-host](https://github.com/microsoft/agent-framework/tree/main/python/samples/04-hosting/durabletask)

::: zone-end

## Next steps

- [Durable Task for AI agents overview](./durable-task-for-ai-agents.md)
- [Agentic application patterns](./durable-agents-patterns.md)
- [Azure Functions (Durable)](/agent-framework/integrations/azure-functions)
- [Durable Task Scheduler overview](../scheduler/durable-task-scheduler.md)
