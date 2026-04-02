---
title: Connectors in Azure SRE Agent
description: Extend your agent's capabilities to external data sources, collaboration tools, and custom APIs using connectors.
ms.topic: concept-article
ms.service: azure-sre-agent
ms.date: 03/30/2026
author: craigshoemaker
ms.author: cshoe
ms.ai-usage: ai-assisted
ms.custom: connectors, integrations, mcp, outlook, teams, data sources, status, health, monitoring, auto-recovery, wildcard, mcp tools
#customer intent: As an SRE, I want to connect my agent to external systems so that it can query data, take actions, and collaborate across my toolchain.
---

# Connectors in Azure SRE Agent

<video 
  controls 
  style={{width: '100%', maxWidth: '800px', marginTop: '1rem', marginBottom: '1.5rem', borderRadius: '8px'}}
>
  <source src={useBaseUrl('/video/From_Chat_to_Action.mp4')} type="video/mp4" />
  Your browser does not support the video tag.
</video>

Your agent comes with built-in access to Azure services—it can query Azure Monitor, Application Insights, Log Analytics, and Azure Resource Graph out of the box. Connectors extend that reach to external systems: your Kusto clusters, source code repositories, collaboration tools, and custom APIs.

![Diagram showing how connectors bridge the agent to external systems like GitHub, Kusto, Teams, and custom APIs](/img/diagrams/connectors-flow.svg)

:::note Connectors vs. incident platforms
**Connectors** give your agent access to data and actions—querying logs, sending notifications, reading code. **Incident platforms** are a separate concept: they control where alerts come FROM and how your agent responds to them automatically.

This page covers connectors. For incident platforms, see [Incident platforms](./incident-platforms).

## What your agent can do without connectors

Even with no connectors configured, your agent has built-in capabilities through its managed identity and Azure RBAC permissions:

| Built-in capability | What it provides |
|---------------------|-----------------|
| **Application Insights** | Query application telemetry, traces, and exceptions |
| **Log Analytics** | Query Log Analytics workspaces |
| **Azure Monitor metrics** | List and query metrics, analyze trends and anomalies |
| **Azure Resource Graph** | Discover and query any Azure resource across subscriptions |
| **ARM / Azure CLI** | Read and modify any Azure resource type |
| **AKS diagnostics** | Run kubectl commands, diagnose Kubernetes issues |

Azure Resource Graph and ARM operations work with **any Azure resource type**—App Services, Container Apps, VMs, networking, storage, and more. If your logs and metrics live in Azure Monitor and Application Insights, your agent can start investigating issues immediately—no connector setup required. Connectors become valuable when you need the agent to reach systems *outside* Azure.

## What connectors provide

Connectors fall into four categories based on what they give your agent:

### Data sources

Query logs, metrics, and telemetry stored outside Azure Monitor.

| Connector | What it provides |
|-----------|-----------------|
| **Database query (Azure Data Explorer)** | Run predefined KQL queries against your Kusto clusters |
| **Database indexing (Azure Data Explorer)** | Auto-learn your Kusto schema so the agent can generate queries dynamically |

### Source code and knowledge

Give your agent context about your systems—code, wikis, and documentation.

| Connector | What it provides |
|-----------|-----------------|
| **GitHub MCP server** | Access to repositories, issues, pull requests, and wiki pages |
| **GitHub OAuth** | GitHub access via OAuth authentication flow |
| **Azure DevOps OAuth** | Azure DevOps access via OAuth authentication |
| **Documentation (Azure DevOps)** | Index and search your Azure DevOps wikis |

With these connectors, your agent can search code for error patterns, read wiki documentation, reference API docs during troubleshooting, and connect incidents to related pull requests.

### Collaboration tools

Let your agent communicate findings through the channels your team already uses.

| Connector | What it provides |
|-----------|-----------------|
| **Send notification (Teams)** | Post findings and updates to Teams channels |
| **Send email (Outlook)** | Email investigation summaries and reports |

### Custom connectors (MCP servers)

MCP (Model Context Protocol) lets you connect your agent to any system — observability platforms, source code repositories, ticketing systems, and custom APIs. Your agent auto-discovers tools from connected servers, monitors connection health with 60-second heartbeats, and recovers from transient failures automatically.

Two transport types cover every deployment model: **Streamable-HTTP** for remote cloud services, and **stdio** for local processes running alongside your agent. Pre-configured partner connectors for GitHub, Datadog, Splunk, New Relic, and more provide one-click setup.

For a complete guide to MCP architecture, transport types, partner connectors, health monitoring, and tool management, see [MCP Connectors & Tools(mcp-connectors.md).

To set up your first MCP connector, see [Set up MCP connector](setup-mcp-connector.md).

## Browsing and managing connectors

Open the Connectors page (**Builder > Connectors**) to see your connectors organized into collapsible category groups. All groups are expanded by default.

| Category | What it includes |
|----------|-----------------|
| **Code Repository** | GitHub, Azure DevOps, source code and documentation connectors |
| **Notification** | Teams and Outlook messaging connectors |
| **Telemetry** | Azure Data Explorer, Datadog, Dynatrace, Elasticsearch, New Relic, Splunk, and other monitoring connectors |
| **Other** | Generic MCP servers and connectors that don't fit other categories |

<img src="/img/concepts/connectors-grouped-expanded.png" alt="Connectors page showing category groups expanded with connector count badges" style={{maxWidth: '80%'}} />

Each category header shows the number of connectors in that group. When you collapse a category, a red badge appears if any connector in that group has a connection issue — so you can spot problems at a glance without expanding every section.

<img src="/img/concepts/connectors-grouped-collapsed.png" alt="Collapsed connector category showing count badge and failed connector indicator" style={{maxWidth: '80%'}} />

Use the toolbar controls to manage your view:

- **Expand all / Collapse all** — Toggle all category groups at once
- **Category filter** — Show only connectors in a specific category
- **Search** — Find connectors by name (switches to a flat list for keyword lookup)

Only categories that contain at least one connector are displayed. When you search for a connector by name, the page switches to a flat list view for faster filtering.

## Who can configure connectors

Connector management requires **write** permission on the agent. In practice:

| Role | Can configure connectors? |
|------|--------------------------|
| **SRE Agent Administrator** | Yes |
| **SRE Agent Standard User** | No—view only |
| **SRE Agent Reader** | No—view only |

During setup, some connectors require **OAuth consent** from a user who has the appropriate permissions in the external system (for example, a GitHub org member for GitHub connectors, or an Azure AD admin for Outlook/Teams). This consent is about permissions in the *external* service, not SRE Agent roles.

For connectors that use the agent's **managed identity** (like Azure Data Explorer), an admin of the external system must allowlist the identity.

Once configured, all agent users benefit from connectors automatically—they just ask the agent questions and it uses the available connectors behind the scenes.

## Connectors and custom agents

You can assign specific MCP tools to specialized custom agents. A database troubleshooting custom agent might get Kusto tools, while a deployment custom agent gets GitHub access. This keeps each custom agent focused and prevents overwhelming it with too many tools.

Assign tools individually in the portal tool picker, or use wildcard patterns (`connection-id/*`) in YAML to add all tools from a server at once. For details on tool assignment and wildcard syntax, see [MCP Connectors & Tools(mcp-connectors#how-tools-reach-your-agents.md).

Learn more: [Custom Agents](./subagents)

## Related

| Resource | Why it matters |
|----------|-------------------|
| [Incident platforms](./incident-platforms) | How your agent receives and responds to incidents automatically |
| [Connect source code](connect-source-code.md) | Set up GitHub or Azure DevOps connectors |
| [Set up an MCP connector](setup-mcp-connector.md) | Add custom MCP servers |
| [Custom Agents](./subagents) | Create specialized agents with focused connector access |
| [Permissions](./permissions) | Configure Azure resource access for your agent |
