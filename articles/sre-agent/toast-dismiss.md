---
title: 'Dismiss Notification Toasts'
description: 'Dismiss portal notification toasts immediately instead of waiting for auto-dismiss. Stay focused during rapid workflows with instant toast control.'
author: dchelupati_microsoft
ms.author: dchelupati
ms.date: 03/23/2026
ms.topic: concept-article
ms.service: azure-sre-agent
ai-usage: ai-assisted
---

# Dismiss notification toasts

> [!TIP]
> - Every notification toast includes a **Dismiss** link—click to close immediately.
> - No more waiting for the 4-second auto-dismiss timeout during rapid workflows.
> - Hover over a toast to pause the auto-dismiss timer and read the full message.
> - Works on all toast types: success, error, warning, and info.

## The problem

The Azure SRE Agent portal shows notification toasts when you complete actions—creating an agent, saving settings, submitting feedback. Previously, every toast disappeared only after a fixed timeout. During rapid workflows—deleting multiple agents, updating several connectors, saving configuration changes—toasts stacked up in the top-right corner with no way to clear them. You had to wait for each one to disappear on its own.

## How toast dismiss works

Every notification toast includes a **Dismiss** link positioned to the right of the toast title. Click it to close the toast immediately.

The dismiss and auto-dismiss work independently:

| Behavior | What happens |
|----------|-------------|
| **Click Dismiss** | Toast closes immediately |
| **Wait** | Toast auto-dismisses after 4 seconds |
| **Hover** | Auto-dismiss timer pauses while your cursor is over the toast |
| **Switch windows** | Auto-dismiss timer pauses until you return |

Toasts appear in the top-right corner of the portal, below the navigation bar.

## What makes this different

Unlike waiting for auto-dismiss, clicking **Dismiss** lets you clear notifications as fast as you work. When you create three agents in a row, success confirmations don't linger on screen.

You can also review past notifications at any time. Select the **Notifications** bell icon in the navigation bar to open the notification drawer. The drawer keeps a history of all notifications—each with its own dismiss button—so you never lose track of what happened while you were focused elsewhere.

## Before and after

| Before | After |
|--------|-------|
| Toasts auto-dismiss after 4 seconds—no manual close option | Select **Dismiss** to close any toast immediately |
| Multiple toasts stack up during rapid actions | Clear completed notifications as fast as you work |
| Wait for timeout to see content behind the toast | One selection to restore your view |
| Hover pauses the timer, but you still have to wait | Hover to read, then dismiss when ready |

## Toast notification types

Every portal action that shows a confirmation or status message uses the same toast system. All toast types include the Dismiss link:

| Toast type | When it appears | Example |
|-----------|----------------|---------|
| **Info** | Feedback, general confirmations | "Thank you for your feedback!" |
| **Success** | Completed operations | Agent created, connector saved |
| **Error** | Failed operations | Subscription load errors |
| **Warning** | Cautionary notices | Validation warnings |

## Get started

Toast dismiss requires no configuration — every toast in the portal includes the **Dismiss** link automatically. The next time a notification toast appears, look for the **Dismiss** link to the right of the message title.

## Related content

- [Send notifications](send-notifications.md)
- [Audit agent actions](audit-agent-actions.md)
- [Workflow automation](automate-workflows.md)
