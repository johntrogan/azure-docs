---
title: ADX cluster grouping in Azure SRE Agent
description: Connect multiple Azure Data Explorer clusters through a single connector using identity-based groups instead of one connector per cluster.
ms.topic: feature-guide
ms.service: azure-sre-agent
ms.date: 03/23/2026
author: craigshoemaker
ms.author: cshoe
ms.ai-usage: ai-assisted
ms.custom: kusto, adx, azure-data-explorer, cluster-grouping, connectors, managed-identity, multi-cluster, telemetry
#customer intent: As an SRE, I want to group multiple ADX clusters under a single connector so that I can manage telemetry connections without connector sprawl.
---

# ADX cluster grouping in Azure SRE Agent

> [!TIP]
> - **One connector per identity group** - Connect multiple ADX clusters with a single connector instead of creating one per cluster.
> - **Per-cluster health status** - See which clusters are healthy and which need attention, individually.
> - **Group management** - Organize clusters by identity, add or remove clusters without recreating connectors.
> - **Parallel connection testing** - All clusters in a group are tested simultaneously during setup.

## The problem: connector sprawl

Teams with telemetry spread across multiple Azure Data Explorer clusters—by region, by service, or by team—end up creating a separate connector for each cluster. A team with eight regional clusters needs eight connectors, all configured with the same managed identity, all appearing as separate rows in the Connectors list.

This creates three problems:

1. **Setup overhead** - Each cluster means walking through the full connector wizard again with the same identity configuration.
1. **No logical grouping** - The Connectors list shows eight flat rows with no way to see that they belong together.
1. **Scattered health status** - If one cluster in a logical group has a connectivity issue, you scan all eight rows to find it.

## How cluster grouping works

The ADX connector wizard lets you create **one connector per managed identity group** with multiple cluster URIs. Instead of creating separate connectors, you add all your cluster URIs to a single connector and organize them into groups.

:::image type="content" source="media/adx-cluster-grouping/step-03-add-clusters-form.png" alt-text="Screenshot showing the Add clusters step with group name, managed identity dropdown, and cluster URI fields.":::

The four-step wizard guides you through:

1. **Choose a connector** - Select Azure Data Explorer from the Telemetry tab.
1. **Set up connector** - Enter a name and select the managed identity.
1. **Add clusters** - Define cluster groups with URIs and optional per-group identity overrides.
1. **Review + test connection** - Test all clusters in parallel and see per-cluster results.

Each cluster group has:

| Field | Description |
|-------|-------------|
| **Group name** | A label for this group (default: "ClusterGroup"). |
| **Managed identity** | Defaults to "(inherit)" from the connector-level identity. Override with a specific identity per group. |
| **Cluster URIs** | One or more URIs in format `https://<CLUSTER>.<REGION>.kusto.windows.net/<DATABASE>`. New rows appear automatically as you fill existing ones. |

Select **+ Create new group** to add additional groups with different identity assignments.

> [!TIP]
> Each cluster URI must include the database name in the path—for example, `https://mycluster.westus.kusto.windows.net/mydb`. A URI without a database path is rejected during validation.

### Per-cluster connection testing

When you select **Test connection** on the review step, the wizard runs a lightweight query against each cluster in parallel:

| Icon | Meaning |
|------|---------|
| Green checkmark | Cluster connected and responding |
| Red dismiss icon | Cluster failed—error message shown alongside |
| "Not tested" badge | Test hasn't run yet |

:::image type="content" source="media/adx-cluster-grouping/step-04-review-before-test.png" alt-text="Screenshot showing the Review and test connection step with connector summary and Test connection button.":::

The button text follows a two-click flow:

1. Select **Test connection**—runs tests, shows per-cluster results.
1. Select **Add connector**—saves the connector (only appears after testing completes).

### Health monitoring after save

Health checks continue to report per-cluster status after the connector is saved. If some clusters become unreachable, the connector status calls out the failing clusters by name—for example, *"2 cluster(s) failed: cluster1, cluster2"*—no need to check individual connector rows.

The connector appears in your Connectors list under the **Telemetry** category.

## What makes this different

**Unlike one-connector-per-cluster**, cluster grouping reflects how teams actually organize their telemetry. Clusters that share an identity belong together—the connector model now matches that reality.

**Unlike manual tracking**, the per-cluster health check surfaces problems at the individual cluster level. When one cluster in a group of eight has a permission issue, you see exactly which one failed—not a generic "connector unhealthy" status across eight separate rows.

**Unlike recreating connectors**, you can edit an existing grouped connector to add or remove cluster URIs without starting over.

## Before and after

| | Before | After |
|---|--------|-------|
| **Connectors for 8 clusters** | 8 separate connectors, identical identity | 1 connector with 8 cluster URIs |
| **Adding a new cluster** | Full wizard walkthrough | Edit existing connector, add one URI |
| **Finding a failing cluster** | Scan all 8 connector rows | Per-cluster status in one view |
| **Identity changes** | Update 8 connectors individually | Update one connector's identity |
| **Connectors list** | 8 flat rows under Telemetry | 1 row representing the whole group |

## Multiple identity groups

If your clusters use different managed identities—for example, one set of clusters in a production tenant and another in a staging tenant—use the **+ Create new group** button to add additional groups within the same connector.

Each group can override the connector-level identity with its own managed identity. The "(inherit)" option uses whatever identity you selected in step 2.

:::image type="content" source="media/adx-cluster-grouping/step-03-add-clusters-empty.png" alt-text="Screenshot showing the Add clusters step with one cluster URI filled and an auto-added empty row.":::

## Example: consolidating regional telemetry clusters

Your team runs services across three Azure regions, each with its own ADX cluster:

| Cluster | Database | Region |
|---------|----------|--------|
| `https://prod-westus.westus.kusto.windows.net/servicetelemetry` | servicetelemetry | West US |
| `https://prod-eastus.eastus.kusto.windows.net/servicetelemetry` | servicetelemetry | East US |
| `https://prod-westeu.westeurope.kusto.windows.net/servicetelemetry` | servicetelemetry | West Europe |

With cluster grouping, you create one connector named `prod-telemetry`, select your managed identity in the setup step, then add all three cluster URIs in a single group called "Production Regions." After testing confirms all three clusters connect, your agent can query telemetry from any region through a single connector.

## Edit grouped connectors

To add or remove clusters from an existing connector:

1. Select the connector's **...** menu > **Edit**.
1. The edit dialog opens with the connector name locked but cluster configuration editable.
1. Add new cluster URIs, remove existing ones, or adjust group identities.
1. Save your changes.

## Existing connectors

Connectors created before cluster grouping continue to work. They aren't migrated automatically. To consolidate, create a new grouped connector with all your cluster URIs, verify all clusters connect, and remove the old individual connectors.

> [!NOTE]
> New connectors use the grouped architecture by default.

## Related content

- [Kusto tools](kusto-tools.md)
- [Connectors overview](connectors.md)
- [Set up an Azure Data Explorer connector](kusto-connector.md)
