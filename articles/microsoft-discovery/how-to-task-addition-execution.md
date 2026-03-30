---
title: Task addition and execution
description: Learn how to add tasks to an investigation, manage task relationships, monitor execution progress, and handle results in the Discovery Engine.
author: hectoralinares
ms.author: hectorl
ms.service: azure
ms.topic: how-to
ms.date: 03/30/2026

#CustomerIntent: As a researcher or scientist, I want to know how to create and manage tasks so that I can effectively guide the Discovery Engine through my research.
---

# Task addition and execution

This guide covers the practical steps for creating tasks, setting up relationships between them, monitoring execution, and managing results in the [Discovery Engine](concept-discovery-engine.md). For conceptual background on task structure and statuses, see [Tasks and workflows](concept-tasks-workflows.md).

## Adding a task

### Using Discovery Studio

1. Open your investigation in Discovery Studio.
2. Select **Add Task**.
3. Fill in the required fields:
   - **Title**: A short, specific name.
   - **Description**: Detailed context for the executing agent.
   - **Validation requirements**: At least one criterion for evaluating the result.
4. Optionally set:
   - **Priority**: High, Medium, or Low.
   - **Depends on**: Select tasks that must complete first.
   - **Parent**: Select the parent task that is part of a larger objective.
5. Select **Save**.

The task appears in your investigation with a status of New. If cognition is enabled, it picks up the task on its next reasoning cycle.

### Adding tasks while cognition is running

You can add tasks to an active investigation at any time. Cognition detects new tasks on its next cycle and evaluates whether they're ready to execute based on their dependencies.

Keep in mind:

- If the new task has no dependencies, cognition might start it immediately.
- If the new task depends on tasks that are already complete, cognition starts it on the next cycle.
- If the new task depends on tasks that are still in progress, cognition waits.

> [!TIP]
> If you're adding several related tasks with dependencies between them, add all the tasks first, then set the dependencies. Setting up dependencies prevents cognition from starting a task before defining its relationships.

## Setting up task relationships

### Dependencies

Dependencies control execution order. A task with a dependency waits until the dependency reaches a terminal status (Complete, Needs User Attention, Failed, Removed, or Stale).

To add a dependency:

1. Open the task you want to add a dependency to.
2. In the **Depends on** field, select the tasks this task needs to wait for.
3. Save the task.

**Common dependency patterns:**

Sequential chain:
```
Task A → Task B → Task C
```
Each task waits for the previous one to finish.

Fan-out (parallel):
```
Task A → Task B
Task A → Task C
```
Tasks B and C both wait for A, then run in parallel.

Fan-in (merge):
```
Task B → Task D
Task C → Task D
```
Task D waits for both B and C to finish before starting.

Diamond (combine fan-out and fan-in):
```
Task A → Task B → Task D
Task A → Task C → Task D
```
A runs first, then B and C in parallel, then D after both complete.

### Parent-child relationships

Parent-child relationships organize tasks into a hierarchy. They don't control execution order on their own. Use dependencies alongside parent-child relationships when you need both structure and sequencing.

To set a parent:

1. Open the child task.
2. In the **Parent** field, select the parent task.
3. Save.

When all child tasks of a parent reach terminal status, cognition can execute the parent task to synthesize the child results into a combined output.

> [!NOTE]
> Setting a parent doesn't automatically create a dependency. If you want the parent task to wait for all children, set explicit dependencies from the parent to each child task.

## Monitoring task execution

### Task statuses

As cognition works, tasks move through statuses. The key statuses to watch for are:

| Status | What's happening |
|--------|-----------------|
| **New** | Waiting to start (dependencies might not be met yet) |
| **Executing** | An agent is actively working on this task |
| **Validating** | Agent finished; cognition is evaluating the result |
| **Complete** | Result passed validation |
| **Incomplete** | Result didn't meet validation requirements; cognition might retry |
| **Needs User Attention** | Cognition couldn't resolve this task after multiple attempts |

For the full status reference, see [Task status lifecycle](concept-tasks-workflows.md#task-status-lifecycle).

### Execution history

Each task maintains a record of every execution attempt. The execution history shows:

- which agent cognition assigned
- When execution started and how long it took
- Whether tools were called and what they returned
- The validation outcome and any comments

To view execution history, open a task and look at the **Execution History** section.

### Validation comments

After each execution attempt, the validation process adds comments to the task explaining whether each validation requirement was met. These comments help you understand:

- What passed and what failed
- Why a task was marked Incomplete
- What cognition tried differently on subsequent attempts

## Managing task results

### Reviewing results

When a task reaches Complete status, its result is available in the task detail view. Results can include:

- Text output from the agent (analysis, summaries, recommendations)
- References to data assets (files, datasets) created during execution
- Structured data (tables, sequences, computed properties)

### Using results in dependent tasks

When Task B depends on Task A, the agent executing Task B has access to Task A's result. You don't need to manually pass data between tasks. Cognition provides the dependency context to the executing agent.

To make this work well:

- Write Task B's description to reference what it should expect from Task A. For example: "Using the protein sequences retrieved in the previous step, compute molecular properties."
- Set explicit dependencies so cognition knows which tasks provide input.

### Handling unsatisfactory results

If a completed task's result doesn't meet your expectations, even though it passed validation:

1. **Adjust the validation requirements** to be more specific about what was missing.
2. **Add a comment** explaining what needs to change.
3. **Change the status** back to New. Cognition retries the task with your updated requirements and comments as more context.

Alternatively, create a new follow-up task that builds on the existing result rather than replacing it.

## Adding tasks created by cognition

When cognition decomposes a broad objective, it creates child tasks automatically. These tasks appear in your investigation alongside the ones you created.

You can:

- **Review and edit** child tasks that cognition created. Adjust descriptions or validation requirements if they don't match your expectations.
- **Remove** child tasks that aren't relevant by setting their status to Removed.
- **Add dependencies** between cognition-created tasks and your own tasks.
- **Add comments** to guide cognition's approach for specific child tasks.

Cognition-created tasks aren't special. They work exactly like tasks you create manually.

## Manually completing tasks

To complete a task, yourself rather than having an agent do it:

1. Open the task.
2. Add your result in the **Result** field.
3. Change the status to **Complete**.

Cognition sees your result and makes it available to any dependent tasks. Useful when you have domain expertise for a specific step, or when you want to provide ground truth data that agents can build on.

## Related content

- [Tasks and workflows](concept-tasks-workflows.md)
- [Build workflows with cognition](how-to-build-workflows-cognition.md)
- [Advanced workflow patterns](concept-advanced-workflow-patterns.md)
- [Debug task execution](how-to-debug-task-execution.md)
