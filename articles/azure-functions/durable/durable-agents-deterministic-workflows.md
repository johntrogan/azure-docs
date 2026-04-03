---
title: Durable Functions and Durable Task SDKs for deterministic agentic workflows - Azure
description: Learn how to build deterministic agentic workflows with any AI framework using Durable Functions or the Durable Task SDKs.
author: greenie-msft
ms.topic: conceptual
ms.date: 03/15/2026
ms.author: nigreenf
---

# Durable Functions and Durable Task SDKs for deterministic agentic workflows

[Durable Functions](./durable-functions-overview.md) and the [Durable Task SDKs](./durable-task-scheduler/durable-task-overview.md) let you build deterministic agentic workflows with any AI framework. You write the workflow logic as ordinary code, including the sequence of steps, branching, parallelism, and error handling, and *durable execution* handles checkpointing, retries, and distributed scaling underneath. It supports .NET, Python, Java, and TypeScript/JavaScript.

## When to use deterministic agentic workflows

In a deterministic agentic workflows, your code controls the execution path. You decide which agents run, in what order, with what inputs, and how outputs and errors are handled using standard programming constructs (`if/else`, loops, `try/catch`). The LLM performs work inside individual steps but doesn't directly decide what happens next.

Deterministic agent workflows contrast with agent-directed workflows. In agent-directed workflows, the Large Language Model (LLM) drives the control flow by handling user inputs, selecting tools, responding to the outputs of those tools, and looping until it decides to stop.

Choose Durable Functions or the Durable Task SDKs for deterministic agent workflows when:

- **You want code-defined control flow.** You write the orchestration logic as ordinary code. Each step, branch, and error path is explicit and reviewable.
- **You want the flexibility to use any AI framework or model API.** The Durable Task programming model is general purpose and can be used with any AI framework.
- **You need broad language support.** The Durable Task programming model is available in .NET, Python, Java, and JavaScript/TypeScript.
- **You're adding AI steps to an existing durable workflow.** If you're already using the Durable Task programming model in your application architecture, adding agentic actions with durable execution guarantees is very straightforward.

## Compare agentic workflow options on Azure

There are many options for building agentic workflows on Azure today, including using the Durable Task programming model (described in this article), [Microsoft Agent Framework (MAF) workflows](/agent-framework/workflows/), and [Logic Apps _agent loop_](../logic-apps/agent-workflows-concepts.md). The following table provides a brief comparison of these options for building agentic workflows to help you decide the technology that best fits your needs.

| Capability | Durable Functions / Durable Task SDKs | Microsoft Agent Framework workflows | Logic Apps Agent Loop |
|---|---|---|---|
| **Control flow** | Code-defined (imperative) | Code-defined (graphs) | Designer / declarative (JSON) |
| **Programming languages** | .NET, Python, Java, TypeScript/JavaScript | .NET, Python | Visual designer / JSON |
| **AI framework support** | Any | Optimized for Microsoft Agent Framework | Built-in AI connectors |
| **Hosting** | Any, with built-in Azure Functions support | Any, with first-class [Foundry Hosted Agents](../../../foundry/agents/concepts/hosted-agents.md) support | Azure Logic Apps managed service (Consumption or Standard SKU) |
| **State storage** | Durable Task Scheduler (managed) | Bring your own (extensible via checkpoint manager) | Logic Apps runtime (managed) |
| **Agent-directed workflows** | Not built-in (deterministic only) | Yes, via the [Durable Task extension for MAF](./durable-agents-microsoft-agent-framework.md) | Yes, via the Agent Loop action |
| **Target audience** | Backend developers | Application developers | Integration developers / low-code users |
| **Long-running tasks** | First-class (hours / days / weeks / eternal) | Supported via developer-controlled workflow state checkpointing | Supported for *stateful* workflows only (up to 90 days) |
| **Recovery from failure** | Automatic | Manual | Automatic |
| **Observability** | Execution history in the Durable Task Scheduler dashboard, OpenTelemetry | OpenTelemetry, custom visualization | Azure Monitor / Logic Apps diagnostics |
For a comparison of Durable Functions vs. the standalone Durable Task SDKs, see [Choose your orchestration framework](./choose-orchestration-framework.md).

The Durable Task programming model maps naturally to agentic patterns: **orchestrations** define the workflow control flow (sequence, branching, parallelism, error handling) and are automatically checkpointed, while **activities** wrap non-deterministic operations like LLM calls, tool invocations, and API requests that can run on any available compute instance.

## Key capabilities for agentic workflows

| Capability | How it applies to agents |
|---|---|
| **Resilient agent calls** | Wrap LLM calls with configurable retry policies (max attempts, backoff strategy) to handle rate limits and transient failures automatically. |
| **Human-in-the-loop** | Pause an orchestration to wait for human approval, user input, or an external signal. The orchestration is unloaded from memory while waiting, incurring zero compute cost. |
| **Sub-agents** | Delegate complex subtasks to child orchestrations. A parent orchestration can spawn specialized agent workflows and aggregate their results. |
| **Parallel agent invocation** | Run multiple agent activities in parallel (for example, querying several models or researching multiple topics simultaneously), then combine the results. |
| **Timeouts and deadlines** | Set timeouts on agent steps or schedule future actions. Combine with external events to implement approval deadlines. |
| **Long-running agent loops** | Create long-running agent loops that periodically check for new work, process queues, or monitor conditions indefinitely. |

## Durable Functions examples

The following examples use the [Durable Functions](./durable-functions-overview.md) programming model, which runs on Azure Functions with serverless hosting.

### Content review with human-in-the-loop (Durable Functions)

This orchestration chains LLM-powered agent activities and includes human-in-the-loop approval. Each activity function wraps an AI agent call, and the orchestration controls the overall flow:

# [C#](#tab/csharp)

```csharp
// Activity function: wraps an AI agent call as a durable activity
[Function(nameof(GenerateContentAgent))]
public async Task<string> GenerateContentAgent(
    [ActivityTrigger] string topic)
{
    // Call any AI framework or model API here:
    //   - Semantic Kernel, AutoGen, LangChain, etc.
    //   - Azure OpenAI SDK, OpenAI SDK, or direct HTTP calls
    string result = await CallYourAgentOrModelAsync(topic);
    return result;
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

# [Python](#tab/python)

```python
# Activity function: wraps an AI agent call as a durable activity
@app.activity_trigger(input_name="topic")
def generate_content_agent(topic: str) -> str:
    # Call any AI framework or model API here:
    #   - Semantic Kernel, LangChain, AutoGen, etc.
    #   - Azure OpenAI SDK, OpenAI SDK, or direct HTTP calls
    result = call_your_agent_or_model(topic)
    return result

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

# [JavaScript](#tab/javascript)

```javascript
const df = require("durable-functions");

// Activity function: wraps an AI agent call as a durable activity
df.app.activity("generateContentAgent", {
    handler: async function (topic) {
        // Call any AI framework or model API here:
        //   - Semantic Kernel, LangChain, AutoGen, etc.
        //   - Azure OpenAI SDK, OpenAI SDK, or direct HTTP calls
        const result = await callYourAgentOrModel(topic);
        return result;
    },
});

// Orchestration: controls the overall workflow
df.app.orchestration("contentReviewOrchestration", function* (context) {
    const topic = context.df.getInput();

    // Step 1: Call an AI agent to generate content
    const draft = yield context.df.callActivity("generateContentAgent", topic);

    // Step 2: Wait for human approval (zero compute cost while waiting)
    yield context.df.callActivity("notifyReviewer", draft);
    const approval = yield context.df.waitForExternalEvent("ApprovalEvent");

    // Step 3: Call an AI agent to refine, or reject
    if (approval.approved) {
        return yield context.df.callActivity("publishContentAgent", draft);
    }

    return { status: "rejected", feedback: approval.feedback };
});
```

---

### Parallel research with fan-out/fan-in (Durable Functions)

This orchestration runs multiple AI agent activities in parallel, then aggregates the results:

# [C#](#tab/csharp)

```csharp
[Function(nameof(ParallelResearchOrchestration))]
public async Task<string> ParallelResearchOrchestration(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var request = context.GetInput<ResearchRequest>();

    // Fan-out: research multiple subtopics in parallel
    var researchTasks = request.Subtopics
        .Select(subtopic => context.CallActivityAsync<string>(
            nameof(ResearchSubtopicAgent), subtopic))
        .ToList();
    string[] researchResults = await Task.WhenAll(researchTasks);

    // Aggregate: synthesize all research into a single summary
    string summary = await context.CallActivityAsync<string>(
        nameof(SynthesizeAgent),
        new { request.Topic, Research = researchResults });

    return summary;
}
```

# [Python](#tab/python)

```python
@app.orchestration_trigger(context_name="context")
def parallel_research_orchestration(context: DurableOrchestrationContext):
    request = context.get_input()

    # Fan-out: research multiple subtopics in parallel
    research_tasks = []
    for subtopic in request["subtopics"]:
        research_tasks.append(
            context.call_activity("research_subtopic_agent", subtopic)
        )
    research_results = yield context.task_all(research_tasks)

    # Aggregate: synthesize all research into a single summary
    summary = yield context.call_activity("synthesize_agent", {
        "topic": request["topic"],
        "research": research_results
    })

    return summary
```

# [JavaScript](#tab/javascript)

```javascript
const df = require("durable-functions");

df.app.orchestration("parallelResearchOrchestration", function* (context) {
    const request = context.df.getInput();

    // Fan-out: research multiple subtopics in parallel
    const tasks = request.subtopics.map((subtopic) =>
        context.df.callActivity("researchSubtopicAgent", subtopic)
    );
    const researchResults = yield context.df.Task.all(tasks);

    // Aggregate: synthesize all research into a single summary
    const summary = yield context.df.callActivity("synthesizeAgent", {
        topic: request.topic,
        research: researchResults,
    });

    return summary;
});
```

---

## Durable Task SDK examples

The following examples use the portable [Durable Task SDKs](./durable-task-scheduler/durable-task-overview.md), which run on any host (Azure Container Apps, Kubernetes, VMs, or locally).

### Content review with human-in-the-loop (Durable Task SDKs)

# [C#](#tab/csharp)

```csharp
using Microsoft.DurableTask;

// Activity: wraps an AI agent call as a durable activity
[DurableTask]
public class GenerateContentAgent : TaskActivity<string, string>
{
    public override Task<string> RunAsync(TaskActivityContext context, string topic)
    {
        // Call any AI framework or model API here:
        //   - Semantic Kernel, AutoGen, LangChain, etc.
        //   - Azure OpenAI SDK, OpenAI SDK, or direct HTTP calls
        return CallYourAgentOrModelAsync(topic);
    }
}

// Orchestration: controls the overall workflow
[DurableTask]
public class ContentReviewOrchestration : TaskOrchestrator<string, string>
{
    public override async Task<string> RunAsync(
        TaskOrchestrationContext context, string topic)
    {
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
}
```

# [Python](#tab/python)

```python
from durabletask import task

# Activity: wraps an AI agent call as a durable activity
def generate_content_agent(ctx: task.ActivityContext, topic: str) -> str:
    # Call any AI framework or model API here:
    #   - Semantic Kernel, LangChain, AutoGen, etc.
    #   - Azure OpenAI SDK, OpenAI SDK, or direct HTTP calls
    return call_your_agent_or_model(topic)

# Orchestration: controls the overall workflow
def content_review_orchestration(ctx: task.OrchestrationContext, topic: str) -> dict:
    # Step 1: Call an AI agent to generate content
    draft = yield ctx.call_activity(generate_content_agent, input=topic)

    # Step 2: Wait for human approval (zero compute cost while waiting)
    yield ctx.call_activity(notify_reviewer, input=draft)
    approval = yield ctx.wait_for_external_event("ApprovalEvent")

    # Step 3: Call an AI agent to refine, or reject
    if approval["approved"]:
        return (yield ctx.call_activity(publish_content_agent, input=draft))

    return {"status": "rejected", "feedback": approval["feedback"]}
```

# [JavaScript](#tab/javascript)

```typescript
import {
  OrchestrationContext,
  TOrchestrator,
  ActivityContext,
} from "@microsoft/durabletask-js";

// Activity: wraps an AI agent call as a durable activity
const generateContentAgent = async (
  _ctx: ActivityContext,
  topic: string
): Promise<string> => {
  // Call any AI framework or model API here:
  //   - Semantic Kernel, LangChain, AutoGen, etc.
  //   - Azure OpenAI SDK, OpenAI SDK, or direct HTTP calls
  return await callYourAgentOrModel(topic);
};

// Orchestration: controls the overall workflow
const contentReviewOrchestration: TOrchestrator = async function* (
  ctx: OrchestrationContext,
  topic: string
): any {
  // Step 1: Call an AI agent to generate content
  const draft: string = yield ctx.callActivity(generateContentAgent, topic);

  // Step 2: Wait for human approval (zero compute cost while waiting)
  yield ctx.callActivity(notifyReviewer, draft);
  const approval = yield ctx.waitForExternalEvent("ApprovalEvent");

  // Step 3: Call an AI agent to refine, or reject
  if (approval.approved) {
    return yield ctx.callActivity(publishContentAgent, draft);
  }

  return { status: "rejected", feedback: approval.feedback };
};
```

# [Java](#tab/java)

```java
import com.microsoft.durabletask.*;
import com.microsoft.durabletask.azuremanaged.DurableTaskSchedulerWorkerExtensions;

DurableTaskGrpcWorker worker = DurableTaskSchedulerWorkerExtensions.createWorkerBuilder(connectionString)
    .addOrchestration(new TaskOrchestrationFactory() {
        @Override
        public String getName() { return "ContentReviewOrchestration"; }

        @Override
        public TaskOrchestration create() {
            return ctx -> {
                String topic = ctx.getInput(String.class);

                // Step 1: Call an AI agent to generate content
                String draft = ctx.callActivity(
                    "GenerateContentAgent", topic, String.class).await();

                // Step 2: Wait for human approval (zero compute cost while waiting)
                ctx.callActivity("NotifyReviewer", draft, Void.class).await();
                ApprovalResponse approval = ctx.waitForExternalEvent(
                    "ApprovalEvent", ApprovalResponse.class).await();

                // Step 3: Call an AI agent to refine, or reject
                if (approval.isApproved()) {
                    String result = ctx.callActivity(
                        "PublishContentAgent", draft, String.class).await();
                    ctx.complete(result);
                } else {
                    ctx.complete("Rejected: " + approval.getFeedback());
                }
            };
        }
    })
    // Activity: wraps an AI agent call as a durable activity
    .addActivity(new TaskActivityFactory() {
        @Override
        public String getName() { return "GenerateContentAgent"; }

        @Override
        public TaskActivity create() {
            return ctx -> {
                String topic = ctx.getInput(String.class);
                // Call any AI framework or model API here:
                //   - Semantic Kernel, LangChain, AutoGen, etc.
                //   - Azure OpenAI SDK, OpenAI SDK, or direct HTTP calls
                return callYourAgentOrModel(topic);
            };
        }
    })
    .build();
```

---

### Parallel research with fan-out/fan-in (Durable Task SDKs)

# [C#](#tab/csharp)

```csharp
using Microsoft.DurableTask;

[DurableTask]
public class ParallelResearchOrchestration : TaskOrchestrator<ResearchRequest, string>
{
    public override async Task<string> RunAsync(
        TaskOrchestrationContext context, ResearchRequest request)
    {
        // Fan-out: research multiple subtopics in parallel
        var researchTasks = request.Subtopics
            .Select(subtopic => context.CallActivityAsync<string>(
                nameof(ResearchSubtopicAgent), subtopic))
            .ToList();
        string[] researchResults = await Task.WhenAll(researchTasks);

        // Aggregate: synthesize all research into a single summary
        string summary = await context.CallActivityAsync<string>(
            nameof(SynthesizeAgent),
            new { request.Topic, Research = researchResults });

        return summary;
    }
}
```

# [Python](#tab/python)

```python
from durabletask import task

def parallel_research_orchestration(ctx: task.OrchestrationContext, request: dict) -> str:
    # Fan-out: research multiple subtopics in parallel
    research_tasks = []
    for subtopic in request["subtopics"]:
        research_tasks.append(
            ctx.call_activity(research_subtopic_agent, input=subtopic)
        )
    research_results = yield task.when_all(research_tasks)

    # Aggregate: synthesize all research into a single summary
    summary = yield ctx.call_activity(synthesize_agent, input={
        "topic": request["topic"],
        "research": research_results
    })

    return summary
```

# [JavaScript](#tab/javascript)

```typescript
import {
  OrchestrationContext,
  TOrchestrator,
  whenAll,
} from "@microsoft/durabletask-js";

const parallelResearchOrchestration: TOrchestrator = async function* (
  ctx: OrchestrationContext,
  request: { topic: string; subtopics: string[] }
): any {
  // Fan-out: research multiple subtopics in parallel
  const tasks = request.subtopics.map((subtopic) =>
    ctx.callActivity(researchSubtopicAgent, subtopic)
  );
  const researchResults: string[] = yield whenAll(tasks);

  // Aggregate: synthesize all research into a single summary
  const summary: string = yield ctx.callActivity(synthesizeAgent, {
    topic: request.topic,
    research: researchResults,
  });

  return summary;
};
```

# [Java](#tab/java)

```java
import com.microsoft.durabletask.*;
import com.microsoft.durabletask.azuremanaged.DurableTaskSchedulerWorkerExtensions;
import java.util.List;
import java.util.stream.Collectors;

DurableTaskGrpcWorker worker = DurableTaskSchedulerWorkerExtensions.createWorkerBuilder(connectionString)
    .addOrchestration(new TaskOrchestrationFactory() {
        @Override
        public String getName() { return "ParallelResearchOrchestration"; }

        @Override
        public TaskOrchestration create() {
            return ctx -> {
                ResearchRequest request = ctx.getInput(ResearchRequest.class);

                // Fan-out: research multiple subtopics in parallel
                List<Task<String>> tasks = request.getSubtopics().stream()
                    .map(subtopic -> ctx.callActivity(
                        "ResearchSubtopicAgent", subtopic, String.class))
                    .collect(Collectors.toList());
                List<String> researchResults = ctx.allOf(tasks).await();

                // Aggregate: synthesize all research into a single summary
                String summary = ctx.callActivity(
                    "SynthesizeAgent", researchResults, String.class).await();

                ctx.complete(summary);
            };
        }
    })
    .build();
```

---

## Next steps

- [Durable Task extension for Microsoft Agent Framework](./durable-agents-microsoft-agent-framework.md) — Durable agents with Microsoft Agent Framework
- [Durable Task Scheduler overview](./durable-task-scheduler/durable-task-scheduler.md) — Architecture, features, and setup
- [Choose your orchestration framework](./choose-orchestration-framework.md) — Compare Durable Functions and Durable Task SDKs
