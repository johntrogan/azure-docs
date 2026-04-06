---
title: 'Subscription Permission Visibility'
description: 'See all your Azure subscriptions in the assignment picker with PIM-aware role detection and tenant-level permission inheritance.'
author: dchelupati
ms.author: dchelupati
ms.date: 03/25/2026
ms.topic: concept-article
ms.service: azure-sre-agent
ai-usage: ai-assisted
---

# Subscription permission visibility

All subscriptions appear in the assignment picker—including those requiring elevated permissions. Your current role on each subscription is shown in the "User role" column, updated in real time with PIM-accurate data. Roles assigned at tenant root or management group level correctly inherit to child subscriptions.

> [!TIP]
> - All subscriptions visible, including those you can't assign
> - Your current role is shown with PIM-accurate data—active PIM roles are reflected within seconds
> - Roles at tenant root or management group level inherit to child subscriptions
> - No configuration needed—PIM-aware role detection is always active

## The problem

When you set up your agent's monitoring scope, you select which Azure subscriptions the agent can access. The subscription picker needs to show your current role on each subscription so you know which ones you can assign.

Two challenges made this harder than it should be:

1. **Stale role data**—the picker relied on Azure Resource Graph (ARG) cached data, which can lag behind actual permissions by minutes or hours. If you activated a time-bound role through Privileged Identity Management (PIM)—like activating Owner for an 8-hour window—the picker might still show your previous role (Reader or none), blocking you from selecting subscriptions you legitimately control
1. **Missing inherited roles**—roles assigned at the tenant root or management group level weren't always reflected on child subscriptions, making it look like you had less access than you actually did

The workaround was painful: wait 30+ minutes for the ARG cache to refresh, repeatedly close and reopen the picker, or ask an admin to grant a permanent standing role—defeating the security benefits of just-in-time access.

## How it works

The subscription picker resolves your permissions through a two-phase process designed for both speed and accuracy.

**Phase 1—fast bulk query.** When the picker opens, it fetches all your role assignments across every subscription in a single Azure Resource Graph query. This is fast but may reflect cached data. It also queries your management group hierarchy to detect inherited roles.

**Phase 2—accurate per-subscription verification.** For each subscription visible in the picker, it calls the Azure Resource Manager API to get your current role assignment schedule instances. This returns only currently-active assignments—permanent standing roles plus any active PIM activations. Expired PIM activations are excluded. While this data loads, the role column shows a brief loading indicator that resolves to the accurate role.

The picker organizes your subscriptions into two sections:

| Section | What it shows |
|---------|---------------|
| **Subscriptions you can assign** | Subscriptions where you have Owner or User Access Administrator permissions |
| **Requires Owner or User Access Administrator role** | Subscriptions where you need elevated access—shown with disabled checkboxes |

Each subscription row displays a **User role** column showing your highest-privilege role—Owner, Contributor, Reader, User Access Administrator, or custom roles. If you have no role, the column shows "—".

### How PIM roles are detected

> [!NOTE]
> This feature is in preview.

If your organization uses Privileged Identity Management (PIM) for just-in-time Azure access, the subscription picker detects your active PIM role activations automatically. No configuration is required.

When you activate a time-bound role (like Owner for 8 hours) through PIM, the picker reflects that active role within seconds of opening. If the activated role is Owner or User Access Administrator, the subscription automatically moves from the disabled section to the selectable section.

The detection works with:

1. Direct PIM activations on your user account
1. Group-based PIM activations (roles assigned to a group you're a member of)
1. PIM activations at management group scope (inherited to child subscriptions)

### Role inheritance

Roles assigned at higher scopes inherit to child subscriptions:

1. **Tenant root scope**—a role at `/` applies to every subscription in the tenant
1. **Management group scope**—a role on a management group applies to all subscriptions under that group
1. **Subscription scope**—applies to that specific subscription

When you have roles at multiple levels, the picker shows the highest-privilege role.

## Before and after

| Aspect | Before | After |
|--------|--------|-------|
| **PIM-activated roles** | Showed stale cached role—could lag 30+ minutes | Shows active PIM role within seconds |
| **Tenant/MG-level roles** | Only direct subscription-level roles | Roles from tenant root, management groups, and child scopes inherited |
| **Role loading** | Single cached query | Two-phase: cached query + accurate per-subscription verification |
| **Subscription movement** | Stuck in disabled section until cache refreshed | Moves to selectable section when PIM role confirmed |
| **Group-based roles** | Not always resolved | Group memberships resolved for complete role picture |

## Resolve permission issues

If you see subscriptions in the "Requires Owner or User Access Administrator role" section:

1. Note the subscription name and your current role from the **User role** column
1. **If you have PIM access:** Activate the Owner or User Access Administrator role through [Entra PIM](https://entra.microsoft.com/#view/Microsoft_Azure_PIMCommon/ActivationMenuBlade/~/aadmigratedRoles), then reopen the subscription picker
1. **If you don't have PIM access:** Contact your Azure administrator or the subscription owner to request the required role
1. Return to the subscription picker—the subscription moves to the selectable section

## Where it appears

The subscription picker with PIM-aware role detection appears when you add subscriptions in three places:

1. **During onboarding setup**—"Add subscription" card on the setup page
1. **During the onboarding wizard**—Infrastructure Scope step
1. **After setup**—Settings > Managed resources > "Add subscription"

## Related content

- [Create and set up your agent](create-and-set-up.md)
- [Manage permissions](manage-permissions.md)
- [User roles and permissions](user-roles.md)
