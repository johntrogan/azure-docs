---
title: Model Integration Guide for Microsoft Discovery Agent Tools
description: Learn how to integrate models, code, and external systems into Microsoft Discovery agent workflows using four common patterns.
author: bscallan
ms.author: bscallan
ms.service: azure
ms.topic: conceptual
ms.date: 04/16/2026
---

# Model Integration Guide for Microsoft Discovery

This guide helps you understand how to integrate your models into Microsoft Discovery agent workflows. It walks you through the main integration patterns, when to use each one, and how to get started.

---

## What you’ll learn

By the end of this guide, you’ll be able to:

- Understand the four main integration patterns  
- Choose the right pattern for your scenario  
- Deploy and connect your model or service to an agent  

---

## Prerequisites

Make sure you have:

- An active Azure subscription  
- Access to a Microsoft Foundry project  
- Access to Microsoft Discovery  
- Basic familiarity with Docker and container workflows  

---

## Integration patterns overview

Microsoft Discovery supports four main ways to connect models and tools:

| Pattern | How it works | Where models run | Best for |
|---|---|---|---|
| Container-based client tool (Python) | Run custom code inside a container | Foundry or external endpoint | Custom logic and orchestration |
| Container-based model (Supercomputer) | Run containers in HPC environment | Discovery Supercomputer | Large-scale training/inference |
| OpenAPI-based | Use REST APIs with schema | Foundry or external service | Existing APIs and services |
| MCP-based | Exchange structured context | Foundry or external endpoint | Stateful, multi-step workflows |

---

## Container-based client tool (Python)

Choose this pattern when you need full control over preprocessing, postprocessing, or orchestration.

### How it works

1. Your agent selects the tool  
2. A container is launched with your inputs  
3. Your Python script:
   - Validates inputs  
   - Calls a model endpoint  
   - Applies custom logic  
4. Results are returned to the agent  

### How to set it up

- Deploy a model (e.g., in Foundry or another endpoint)  
- Build a Docker image with your Python code  
- Configure secure access (managed identity recommended)  
- Register the container as a tool in Microsoft Discovery  

---

## Container-based model (Supercomputer)

Choose this when your model needs high-performance compute (HPC), such as large-scale training or heavy inference.

### Key idea

- Your model runs inside a container on the Discovery Supercomputer  
- Agents don’t call it directly—you access it through a tool or service layer  

### How to set it up

- Package your model and dependencies into a container  
- Deploy it to the Discovery Supercomputer  
- Register a tool that interacts with it  

---

## OpenAPI-based

Choose this if you already have a REST API or want a standard, schema-driven integration.

### How it works

- Your model/service is exposed via an HTTP endpoint  
- You define an OpenAPI (Swagger) spec  
- Agents use the spec to discover and call operations  

### How to set it up

- Deploy your model behind an API endpoint  
- Write an OpenAPI specification (requests, responses, auth)  
- Register the API as a tool in Foundry  

---

## MCP-based

Choose this for advanced scenarios that require rich, multi-step, context-aware interactions.

### How it works

- Your tool implements the Model Context Protocol (MCP)  
- Agents send and receive structured context  
- Supports stateful, multi-turn workflows  

### How to set it up

- Deploy your model behind an endpoint  
- Use or build an MCP server  
- Register the MCP tool in Foundry  

---

## How to choose the right pattern

Use this quick guidance:

- Use **container-based tools** → when you need custom logic around models  
- Use **Supercomputer (HPC)** → when compute scale is the main constraint  
- Use **OpenAPI tools** → when you already have REST APIs  
- Use **MCP tools** → when workflows require memory and context across steps  

> [!NOTE]
> You can combine multiple patterns in a single workflow.

---

## Security best practices

No matter which pattern you choose, follow these:

- Use **managed identities and RBAC** (avoid hardcoding credentials)  
- Use **immutable container images** (version and scan them)  
- Use **encryption** (HTTPS + encrypted storage)  
- Enable **logging and monitoring** for all tool and model calls  

---

## What to do next

- Start with a **container-based tool** if you're unsure—it’s the most flexible  
- Move to **OpenAPI or MCP** as your architecture matures  
- Use **HPC deployment** only when scale requires it  

---

## Related content

- [What is Azure AI Foundry?](/azure/foundry/what-is-foundry)
- [Workflows in Microsoft Foundry](/azure/foundry/agents/concepts/workflow)
- [Get started with containers on Azure](/azure/architecture/containers/container-get-started)
- [What is Azure RBAC?](/azure/role-based-access-control/overview)
- [Microsoft OpenAPI documentation](/openapi/)
- [Microsoft Discovery REST API reference](/rest/api/discovery/)