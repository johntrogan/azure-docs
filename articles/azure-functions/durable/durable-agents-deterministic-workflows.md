---
title: Durable Functions and Durable Task SDKs for deterministic agentic workflows - Azure
description: Learn how to build deterministic agentic workflows with any AI framework or direct model API calls using Durable Functions or the Durable Task SDKs.
author: greenie-msft
ms.topic: conceptual
ms.date: 03/15/2026
ms.author: nigreenf
---

# Durable Functions and Durable Task SDKs for deterministic agentic workflows

You don't need to use the Microsoft Agent Framework to benefit from [durable execution](./durable-task-for-ai-agents.md). Using [Durable Functions](./index.yml) or the [Durable Task SDKs](./durable-task-scheduler/durable-task-scheduler.md), you can build deterministic agentic workflows with any AI framework or direct model API calls. This approach gives you full control over orchestration logic and supports .NET, Python, Java, TypeScript/JavaScript, Go, and PowerShell.

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

## Next steps

- [Durable task for AI agents overview](./durable-task-for-ai-agents.md) — What durable execution handles and agentic patterns
- [Durable task extension for Microsoft Agent Framework](./durable-agents-microsoft-agent-framework.md) — Durable agents with Microsoft Agent Framework
- [Durable Task Scheduler overview](./durable-task-scheduler/durable-task-scheduler.md) — Architecture, features, and setup
- [Choose your orchestration framework](./choose-orchestration-framework.md) — Compare Durable Functions and Durable Task SDKs
