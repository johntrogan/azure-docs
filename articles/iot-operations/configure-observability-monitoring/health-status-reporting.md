---
title: Unified Health Status Reporting
description: Learn how to report runtime health status to the cloud using a unified schema.
author: sethmanheim
ms.author: sethm
ms.reviewer: sethm
ms.date: 03/06/2026
ms.topic: concept-article
---

# Observe Azure IoT Operations with unified health and metrics

Azure IoT Operations provides built-in observability to help you understand the health, performance, and behavior of your edge workloads from the cloud. This article explains how unified health status and metrics work together to give you a clear operational view of your AIO deployment.

## Why observability matters

Operators managing Azure IoT Operations clusters need fast, reliable answers to three core questions:

- **Are my services and assets healthy right now?**
- **Is my data flowing as expected?**
- **What is happening inside the system when something goes wrong?**

Azure IoT Operations addresses these needs with cloud-visible health status, metrics, and dashboards that work together to support day-to-day monitoring and troubleshooting.

## Unified health status (current state)

Unified health status provides a simple, real-time signal that shows whether your Azure IoT Operations resources are operating normally.

Azure IoT Operations components running on the edge report health status. The system surfaces this information in:

- Azure Digital Operations Experience (DOE)
- Azure Resource Manager (ARM)
- Azure portal views

### Health states

Each supported resource reports one of the following health states:

|     Status         |     Description                                                         |     Color       |
|--------------------|-------------------------------------------------------------------------|-----------------|
|     Available      |     Resource is healthy and functioning as expected.                   |     🟢 Green     |
| Degraded      | Resource is partially functional but might not operate optimally.    | 🟡 Yellow   |
| Unavailable   | Resource isn't functioning.                                          | 🔴 Red      |
| Unknown       | Health status can't be determined, such as when there are no recent reports. | ⚪ Gray     |

### What health status tells you

Health status answers the question: "Is this resource healthy right now?" It's designed to complement (not replace) provisioning and configuration status:

- **Provisioning status** shows whether a resource was created successfully.
- **Health status** reflects **runtime behavior**, such as pod failures, connectivity issues, or dependency problems.

### Supported resources

The following Azure IoT Operations resources report health status:

- Broker
- Dataflow profiles
- Dataflows and dataflow graphs
- Akri services
- Connectors
- Devices
- Assets

For distributed resources, such as dataflows, devices, and assets, the system aggregates health from multiple instances or subcomponents to provide a single, meaningful status.

### Staleness and freshness

To ensure health data remains trustworthy:

- Components periodically refresh their health status, even when no changes occur.
- If a resource doesn't report health within a defined time window, its status automatically becomes **Unknown**.

This approach prevents stale information from being misinterpreted as healthy.

### Diagnostic details

When a resource is **Degraded** or **Unavailable**, you can access additional information to help you troubleshoot:

- **Reason code** – a stable, documented identifier describing the failure type
- **Message** – a human-readable explanation
- **Timestamps** – when the issue started and when the status was last updated

In DOE and the Azure portal, you can filter and group resources by health state and drill into details for faster investigation.

## Metrics (historical behavior)

While health status shows the current state, metrics provide historical insight into how your system behaves over time.

Azure IoT Operations uses an open, standards-based observability pipeline built on:

- **OpenTelemetry Collector** (running on the edge)
- **Azure Monitor managed service for Prometheus**
- **Azure Managed Grafana**

This pipeline collects and stores metrics emitted by Azure IoT Operations components and makes them available through dashboards.

### What metrics are used for

Metrics help you answer questions like:

- How has throughput changed over the last hour?
- When did error rates start increasing?
- Did latency spike before a failure occurred?

Because metrics are retained over time, they're especially useful for:

- Trend analysis
- Root cause investigation
- Capacity planning

## Unified Grafana dashboard experience

Azure IoT Operations provides a single, unified Grafana dashboard that brings health, metrics, and logs together.

Key characteristics of the dashboard include:

- A health overview, visible at the top.
- Component-specific sections that load only when expanded.
- Metrics and logs shown side-by-side for common troubleshooting workflows.

This design supports the majority of monitoring scenarios without requiring you to jump between tools.

## Metrics documentation and customization

In addition to built-in dashboards, Azure IoT Operations provides documentation for all exposed metrics. This documentation helps you:

- Understand what each metric represents
- Build custom dashboards tailored to your environment
- Extend monitoring to optional components and connectors as needed

## Onboarding observability

Azure IoT Operations simplifies observability setup using a single command to configure the required Azure and edge-side resources.

At a high level, enabling observability:

- Creates or validates Azure Monitor, Grafana, and Log Analytics resources
- Configures the Arc-connected cluster without requiring direct Kubernetes access
- Deploys and configures the OpenTelemetry collector
- Wires Azure IoT Operations components to emit metrics automatically

This approach reduces setup complexity and ensures consistent defaults.

## How health status and metrics work together

Health status and metrics are complementary signals:

| Aspect | Health status | Metrics |
|--------|---------------|---------|
| Purpose | Current state snapshot | Historical trends and patterns |
| Visibility | DOE and Azure portal | Grafana dashboards |
| Use case | "Is my system healthy right now?" | "What happened over the last hour or day?" |

### Example

If a dataflow target becomes unreachable:

- **Metrics** show error counts increasing and throughput dropping.
- **Health status** changes to **Degraded** or **Unavailable** with a reason code.

After recovery:

- **Health status** returns to **Available**.
- **Metrics** preserve the historical record of the incident.

Together, these signals help you detect issues quickly and understand their impact.

## Reason codes for health status

When a resource reports **Degraded** or **Unavailable**, it includes a reason code that identifies the underlying issue. This feature enables faster troubleshooting without needing to immediately dive into logs. The following list shows the possible reason codes with detailed explanations and suggested action items.

| Reason code | Description | Recommended action |
|------------|-------------|--------------------|
| `DataflowTransformSourceSchemaRetrievalFailed` | Failed to retrieve the source schema for a transform. | Verify schema reference and schema registry connectivity. |
| `DataflowTransformTargetSchemaRetrievalFailed` | Failed to retrieve the target schema for a transform. | Verify schema reference and schema registry connectivity. |
| `DataflowTransformConfigurationFailed` | Failed to build the transform pipeline. | Review the dataflow transform configuration. |
| `DataflowTransformEnrichDataFailed` | Failed to enrich data during transform processing. | Check Broker state store connectivity and dataset configuration. |
| `DataflowTransformSourceChannelClosed` | Source input channel closed unexpectedly. | Restart the dataflow pipeline if the issue persists. |
| `DataflowTransformMapperFailed` | One or more transform steps failed during processing. | Review transform configuration and restart the pipeline. |
| `DataflowGraphModuleDownloadFailed` | Failed to download the WASM graph module. | Verify graph artifact availability and registry connectivity. |
| `DataflowGraphModuleDownloadChannelClosed` | Internal channel closed during graph artifact download. | Check pod logs and restart the pod. |
| `DataflowGraphInstantiationFailed` | WASM runtime failed to instantiate the graph. | Verify module compatibility and resource availability; restart the pod. |
| `DataflowGraphConfigurationFailed` | Failed to build the graph pipeline. | Validate graph topology and operation configuration. |
| `DataflowGraphSinkChannelClosed` | Output channel closed unexpectedly. | Restart the pod to recover. |
| `DataflowGraphOutputSendFailed` | Failed to forward processed messages to the target connector. | Check downstream pipeline and restart the pod. |
| `DataflowGraphProcessingFailed` | Unrecoverable error during graph message processing. | Check WASM module logs and restart the pod. |
| `DataflowGraphSourceChannelClosed` | Source input channel disconnected unexpectedly. | Verify source connector health and restart the pod. |
| `DataflowGraphSourceSchemaRetrievalFailed` | Failed to retrieve source schema for the graph. | Verify schema registry connectivity and configuration. |
| `DataflowGraphNoInputHandle` | No input handle found for the graph. | Restart the pod to recover. |
| `DataflowGraphModuleDownloadTimeout` | Graph artifact download didn't complete within the timeout. | Verify artifact existence and controller availability. |
| `DataflowGraphModuleRuntimeFault` | WASM module panicked during execution. | Check pod logs and backtrace; restart the pod. |
| `DataflowMqttSourceConnectionFailed` | MQTT source failed to connect to the broker. | Verify MQTT endpoint, authentication, and network connectivity. |
| `DataflowMqttSourceConfigurationError` | MQTT source failed due to a configuration error. | Review MQTT source configuration. |
| `DataflowMqttSourceSubscriptionFailed` | MQTT subscription request failed. | Verify topic existence and permissions. |
| `DataflowKafkaSourceConnectionFailed` | Kafka source failed to connect to the broker. | Verify broker availability, authentication, and consumer group configuration. |
| `DataflowMqttTargetConfigurationError` | MQTT target encountered a fatal configuration error. | Review target configuration and endpoint settings. |
| `DataflowMqttTargetConnectionFailed` | MQTT target failed to connect to the broker. | Verify endpoint and authentication settings. |
| `DataflowKafkaTargetConfigurationError` | Kafka target failed due to a configuration error. | Review Kafka target configuration. |
| `DataflowKafkaTargetConnectionFailed` | Kafka target failed to connect to the broker. | Verify broker availability and authentication. |
| `DataflowKafkaTargetSendFailed` | Failed to send data to Kafka. | Check broker health and retry behavior. |
| `DataflowKafkaTargetEndpointUnreachable` | Kafka target encountered a fatal client error. | Verify endpoint reachability and restart the client. |
| `DataflowAdxTargetAuthenticationFailed` | Failed to authenticate to Azure Data Explorer. | Verify identity and permissions. |
| `DataflowAdxTargetConfigurationError` | ADX target encountered a configuration or channel error. | Review target configuration and connectivity. |
| `DataflowAdxTargetSendFailed` | Failed to write data to Azure Data Explorer. | Verify ingestion endpoint and retry behavior. |
| `DataflowOtelTargetConfigurationError` | Failed to create OpenTelemetry exporter. | Verify OTEL endpoint and configuration. |
| `DataflowOtelTargetProcessingError` | Failed to process or batch telemetry data. | Review payload format and OTEL configuration. |
| `DataflowOtelTargetSendFailed` | Failed to send telemetry data. | Verify OTEL endpoint reachability. |
| `DataflowObjectStoreTargetAuthenticationFailed` | Authentication failed for object storage. | Verify credentials and permissions. |
| `DataflowObjectStoreTargetConfigurationError` | Object store configuration is missing or invalid. | Review store configuration. |
| `DataflowObjectStoreTargetSendFailed` | Failed to write data to object storage. | Verify endpoint availability and retry behavior. |
| `DataflowDeltaLakeTargetConnectionTimeout` | Delta Lake target connection timed out. | Verify endpoint reachability. |
| `DataflowDeltaLakeTargetAuthenticationFailed` | Delta Lake authentication failed. | Verify identity and permissions. |
| `DataflowDeltaLakeTargetConfigurationError` | Delta Lake configuration is missing or invalid. | Review Delta Lake configuration. |
| `DataflowDeltaLakeTargetSendFailed` | Failed to write data to Delta Lake. | Verify storage endpoint and retry behavior. |

## Next steps

- [Configure observability for your Azure IoT Operations deployment](howto-configure-observability.md)
- [Clean up observability resources](howto-cleanup-observability-resources.md)
