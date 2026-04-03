---
title: Durable Task extension for Microsoft Agent Framework - Azure
description: Learn how to use the Durable Task extension for Microsoft Agent Framework to build fault-tolerant, scalable AI agents with persistent sessions and automatic checkpointing.
author: greenie-msft
ms.topic: conceptual
ms.date: 03/15/2026
ms.author: nigreenf
zone_pivot_groups: azure-durable-approach
---

# Durable Task extension for Microsoft Agent Framework (Preview)

The [Durable Task extension for Microsoft Agent Framework](/agent-framework/integrations/azure-functions) brings [durable execution](./durable-task-for-ai-agents.md) directly into the [Microsoft Agent Framework](/agent-framework/). No changes to your agent logic are required. Register an agent with the extension to make it automatically durable, with persistent sessions, built-in API endpoints, and scaling. 

The extension supports Durable Task's two hosting models: [**Azure Functions (Durable Functions)**](../../azure-functions/durable-functions/durable-functions-overview.md) and the [**Durable Task SDKs**](../sdks/durable-task-overview.md). 

## Single agent orchestration

Define your agent using the standard Microsoft Agent Framework pattern, then host it with the Durable Task extension. The extension handles session persistence, endpoint creation, and state management automatically.

::: zone pivot="durable-functions"

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

::: zone-end

::: zone pivot="durable-task-sdks"

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

::: zone-end

## Multi-agent orchestration

For deterministic multi-agent workflows, coordinate multiple specialized agents as steps in a durable orchestration. Each agent call is checkpointed, and the orchestration recovers automatically if any step fails. Completed agent calls are not re-executed on recovery.

::: zone pivot="durable-functions"

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

::: zone-end

::: zone pivot="durable-task-sdks"

# [Python](#tab/python)

```python
# Orchestration: coordinates multiple agents in a durable workflow
def document_publishing_orchestration(ctx, doc_request: dict):
    research = agent_worker.get_agent(ctx, "ResearchAgent")
    writer = agent_worker.get_agent(ctx, "DocumentPublisherAgent")

    # Step 1: Research the topic
    research_result = yield research.run(
        messages=f"Research the following topic: {doc_request['topic']}",
        response_schema=ResearchResult
    )

    # Step 2: Generate outline from research
    outline = yield ctx.call_activity(generate_outline, input={
        "topic": doc_request["topic"],
        "research_data": research_result.findings
    })

    # Step 3: Write the document
    document = yield writer.run(
        messages=f"""Create a document about {doc_request['topic']}.
        Research findings: {research_result.findings}
        Outline: {outline}""",
        response_schema=DocumentResponse
    )

    # Step 4: Publish
    return (yield ctx.call_activity(publish_document, input={
        "title": doc_request["topic"],
        "content": document.text
    }))
```

# [C#](#tab/csharp)

```csharp
// Orchestration: coordinates multiple agents in a durable workflow
static async Task<string> DocumentPublishingOrchestration(
    TaskOrchestrationContext context, DocumentRequest docRequest)
{
    DurableAIAgent researchAgent = context.GetAgent("ResearchAgent");
    DurableAIAgent writerAgent = context.GetAgent("DocumentPublisherAgent");

    // Step 1: Research the topic
    AgentResponse<ResearchResult> researchResult = await researchAgent
        .RunAsync<ResearchResult>(
            $"Research the following topic: {docRequest.Topic}");

    // Step 2: Generate outline from research
    string outline = await context.CallActivityAsync<string>(
        nameof(GenerateOutline),
        new { docRequest.Topic, researchResult.Result.Findings });

    // Step 3: Write the document
    AgentResponse<DocumentResponse> document = await writerAgent
        .RunAsync<DocumentResponse>(
            $"""Create a document about {docRequest.Topic}.
            Research findings: {researchResult.Result.Findings}
            Outline: {outline}""");

    // Step 4: Publish
    return await context.CallActivityAsync<string>(
        nameof(PublishDocument),
        new { docRequest.Topic, document.Result.Text });
}
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

::: zone pivot="durable-functions"

For complete code samples:
- [.NET Azure Functions](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/AzureFunctions)
- [Python Azure Functions](https://github.com/microsoft/agent-framework/tree/main/python/samples/04-hosting/azure_functions) 

::: zone-end

::: zone pivot="durable-task-sdks"

For complete code samples:
- [.NET any-host](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples/04-hosting/DurableAgents/ConsoleApps)
- [Python any-host](https://github.com/microsoft/agent-framework/tree/main/python/samples/04-hosting/durabletask)

::: zone-end

## Next steps

- [Durable Task for AI agents overview](./durable-task-for-ai-agents.md)
- [Durable Functions and Durable Task SDKs for deterministic agentic workflows](./durable-agents-deterministic-workflows.md)
- [Azure Functions (Durable)](/agent-framework/integrations/azure-functions)
- [Durable Task Scheduler overview](../scheduler/durable-task-scheduler.md)