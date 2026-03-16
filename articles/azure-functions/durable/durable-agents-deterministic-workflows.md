---
title: Durable Functions and Durable Task SDKs for deterministic agentic workflows - Azure
description: Learn how to build deterministic agentic workflows with any AI framework or direct model API calls using Durable Functions or the Durable Task SDKs.
author: greenie-msft
ms.topic: conceptual
ms.date: 03/15/2026
ms.author: nigreenf
---

# Durable Functions and Durable Task SDKs for deterministic agentic workflows

[Durable Functions](./index.yml) and the [Durable Task SDKs](./durable-task-scheduler/durable-task-scheduler.md) let you build deterministic agentic workflows with any AI framework or direct model API calls, giving you full control over orchestration logic. This approach is ideal when you want to integrate [durable execution](./durable-task-for-ai-agents.md) into an existing codebase, use a framework other than Microsoft Agent Framework, or need fine-grained control over every step of the workflow. It supports .NET, Python, Java, TypeScript/JavaScript, Go, and PowerShell.

## When to use this approach

Use Durable Functions or the Durable Task SDKs for agentic workflows when:

- **You need explicit control over the execution path.** You define exactly which agents run, in what order, and with what inputs using standard programming constructs (`if/else`, loops, `try/catch`). The LLM doesn't decide what happens next — your code does.
- **You're using an AI framework other than Microsoft Agent Framework.** Wrap calls to Semantic Kernel, LangChain, AutoGen, or any other framework inside activity functions. The orchestration handles checkpointing and recovery around them.
- **You're calling model APIs directly.** Use the Azure OpenAI SDK, OpenAI SDK, or any HTTP-based model API without an agent framework layer.
- **You need language support beyond .NET and Python.** The Durable Task SDKs support Java, TypeScript/JavaScript, Go, and PowerShell in addition to .NET and Python.
- **You're integrating AI steps into an existing durable workflow.** Add LLM-powered activities to orchestrations that already handle business logic, data processing, or system coordination.

For agent-directed workflows where the LLM decides what to do next (tool selection, conversation flow), and you want durable sessions with minimal code, see [Durable task extension for Microsoft Agent Framework](./durable-agents-microsoft-agent-framework.md).

## How the orchestration model maps to agentic workflows

The durable task programming model has two core building blocks that map naturally to agentic patterns:

- **Orchestrations** control the workflow. They define the sequence of steps, branching logic, parallelism, and error handling. Orchestrations are deterministic and automatically checkpointed — if the process crashes, the orchestration replays from the last checkpoint without re-executing completed steps 
- **Activities** perform the actual work. Each activity wraps a non-deterministic operation: an LLM call, a tool invocation, an API request, or a database write. Activities run on any available compute instance and can be retried independently.

In agentic workflows, the orchestration acts as the **control layer** (deciding what happens and when), while activities act as **agent wrappers** (executing LLM calls, tool invocations, and side effects). This separation means completed LLM calls are never re-executed on recovery, preserving token spend and ensuring consistent results.

### Key capabilities for agentic workflows

| Capability | How it applies to agents |
|---|---|
| **Activity calls with retry policies** | Wrap LLM calls with configurable retry policies (max attempts, backoff strategy) to handle rate limits and transient failures automatically. |
| **External events (human-in-the-loop)** | Pause an orchestration to wait for human approval, user input, or an external signal. The orchestration is unloaded from memory while waiting, incurring zero compute cost. |
| **Sub-orchestrations** | Delegate complex subtasks to child orchestrations. A parent orchestration can spawn specialized agent workflows and aggregate their results. |
| **Fan-out/fan-in** | Run multiple agent activities in parallel (for example, querying several models or researching multiple topics simultaneously), then combine the results. |
| **Durable timers** | Set timeouts on agent steps or schedule future actions. Combine with external events to implement approval deadlines. |
| **Eternal orchestrations** | Create long-running agent loops that periodically check for new work, process queues, or monitor conditions indefinitely. |

## Example: content review with human-in-the-loop

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

## Example: parallel research with fan-out/fan-in

This example demonstrates running multiple AI agent activities in parallel, then aggregating the results — a common pattern when researching a topic from multiple angles or querying different models for comparison.

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

---

## Next steps

- [Durable task for AI agents overview](./durable-task-for-ai-agents.md) — What durable execution handles and agentic patterns
- [Durable task extension for Microsoft Agent Framework](./durable-agents-microsoft-agent-framework.md) — Durable agents with Microsoft Agent Framework
- [Durable Task Scheduler overview](./durable-task-scheduler/durable-task-scheduler.md) — Architecture, features, and setup
- [Choose your orchestration framework](./choose-orchestration-framework.md) — Compare Durable Functions and Durable Task SDKs
