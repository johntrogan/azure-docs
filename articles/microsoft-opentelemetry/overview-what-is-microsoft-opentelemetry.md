---
title: What is Microsoft OpenTelemetry Distro?
description: Learn about the Microsoft OpenTelemetry distribution, a unified observability package for Azure Monitor, OTLP-compatible backends, and A365.
author: AarDavMax
ms.author: aaronmax
ms.topic: overview
ms.service: azure
ms.date: 04/24/2026
ms.custom:
  - template-overview
ROBOTS: NOINDEX
---

# What is Microsoft OpenTelemetry Distro?

Microsoft OpenTelemetry Distro is a unified observability distribution that provides a single onboarding experience for collecting traces, metrics, and logs for agentic and nonagentic applications. It powers observability for Microsoft Agent 365 (A365), Microsoft Foundry, Azure Monitor, or any OTLP-compatible backend. Available for .NET, Node.js, and Python, the distro reduces fragmented setup across multiple observability stacks to one import and one configuration call.

## Key benefits

- **One package, one API**: Replace multiple exporter and instrumentation packages with a single dependency.
- **Multi-backend support**: Send telemetry to Azure Monitor, any OpenTelemetry Protocol (OTLP)-compatible endpoint (such as Datadog, Grafana, or New Relic), and A365 simultaneously.
- **Built-in instrumentations**: Automatic instrumentation for HTTP, databases, Azure SDK, Azure Functions, and more with no extra configuration.
- **Standards-based**: Built on [OpenTelemetry](https://opentelemetry.io/), the industry-standard observability framework.
- **Minimal boilerplate**: One import and one function call in your application entry point is all you need.

## Supported languages

| Language | Package | Install command |
|---|---|---|
| .NET | `Microsoft.OpenTelemetry` | `dotnet add package Microsoft.OpenTelemetry` |
| Python | `microsoft-opentelemetry` | `pip install microsoft-opentelemetry` |
| Node.js | `@microsoft/opentelemetry` | `npm install @microsoft/opentelemetry` |


## How it works

The distro wraps the OpenTelemetry SDK with preconfigured exporters and instrumentations. When you call the setup function, it:

1. Creates standalone OpenTelemetry providers.
1. Attaches Microsoft A365 export when enabled.
1. Configures Azure Monitor export when enabled.
1. Attaches OTLP exporters when an OTLP endpoint is detected.
1. Enables standard instrumentations for HTTP, databases, and frameworks.
1. Enables GenAI instrumentations (OpenAI Agents, LangChain) when configured.

## Backends

### Azure Monitor (Available for Azure Monitor and Foundry)

Export telemetry to Application Insights using a connection string. Includes support for Live Metrics, standard metrics and performance counters.

### OTLP (Recommended for Azure Monitor and Foundry)

Set the `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable and OTLP export is enabled automatically. No code changes needed. Compatible with any OTLP-compliant backend.

### A365

Enable A365 observability to export telemetry to the Microsoft Agent 365 platform, with support for per-request export, baggage propagation, and hosting middleware integration.
