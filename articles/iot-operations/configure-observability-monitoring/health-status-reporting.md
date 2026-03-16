---
title: Unified Health Status Reporting
description: Learn how to report runtime health status to the cloud using a unified schema.
author: sethmanheim
ms.author: sethm
ms.reviewer: vakavali
ms.date: 03/16/2026
ms.topic: concept-article
ai-usage: ai-assisted
---

# Unified health status reporting and observability

Azure IoT Operations provides built-in observability to help you understand the health, performance, and behavior of your edge workloads from the cloud. This article explains how unified health status and metrics work together to give you a clear operational view of your Azure IoT Operations deployment.

## Why observability matters

Operators managing Azure IoT Operations clusters need fast, reliable answers to two core questions:

- **Are my services and assets healthy right now?** Azure IoT Operations now provides a unified health status reporting schema across all components (MQTT brokers, data flows, Akri connectors) and resources (devices, assets). Health status is reported through Azure Resource Manager (ARM) and visible in the operations experience web UI. You can see a simple, cloud-native view indicating whether the system is healthy (green), degraded (yellow), or unhealthy (red).
- **Is my data flowing as expected?** Data flow health is measured by metrics and logs from the cluster, providing a time-series representation of how the system is performing now and historically. This analysis helps predict issues before they impact workloads. For more information, see [Deploy observability resources](howto-configure-observability.md).

Azure IoT Operations addresses these needs with cloud-visible health status, metrics, and dashboards that work together to support day-to-day monitoring and troubleshooting.

## Unified health status (current state)

Unified health status provides a *point-in-time* snapshot of the current health of your Azure IoT Operations components and resources. The system surfaces this information in:

- Operations experience web UI
- Azure Resource Manager (ARM)
- Azure portal views

### Health states

Each supported resource reports one of the following health states:

|     Status    |     Description                                                              |     Color   |
|---------------|------------------------------------------------------------------------------|-------------|
| **Available**     | Resource is healthy and functioning as expected.                             | 🟢 Green    |
| **Degraded**      | Resource is partially functional but might not operate optimally.            | 🟡 Yellow   |
| **Unavailable**   | Resource isn't functioning.                                                  | 🔴 Red      |
| **Unknown**       | Health status can't be determined, such as when there are no recent reports. | ⚪ Gray     |

### How health status is reported

* Components report health status periodically (every minute) to the Kubernetes Custom Resource status field.
* K8s Bridge is a local development tool that syncs status from Kubernetes to Azure Resource Manager, making it visible in the cloud through ARM or the operations experience.
* Each status update includes timestamps (`lastTransitionTime`, `lastUpdateTime`) and optional diagnostic information, such as a message or reason code.
* If a resource doesn't report its status within 15 minutes, it's considered stale and the status is set to **Unknown**.

### What health status tells you

Health status answers the question: "Is this resource healthy right now?" It's designed to complement (not replace) provisioning and configuration status:

- **Provisioning status** shows whether you created and configured the resource successfully.
- **Configuration status** indicates whether the version of the resource was accepted or not.
- **Health status** reflects runtime behavior, such as pod failures, connectivity issues, or dependency problems.

Each Azure IoT Operations and Azure Device Registry resource reports runtime health using a common `healthState` structure.

For example, this structure describes the Kubernetes custom resource status:

```yaml
status:  
  healthState:
    status: Degraded
    lastTransitionTime: "2025-11-03T08:10:12Z"
    lastUpdateTime: "2025-11-03T08:15:00Z"
    message: "Unable to connect to the source endpoint at aio-broker:18883, error code: network unreachable."
    reasonCode: DataflowSourceDisconnected
```

Azure Resource Manager view:

```json
{
  "status": {
    "healthState": {
      "status": "Available",
      "lastTransitionTime": "2026-02-05T20:56:20.078321032+00:00",
      "lastUpdateTime": "2026-02-05T20:56:20.078323363+00:00"
    }
  }
}
```

### Supported resources

The following Azure IoT Operations resources report health status:

- Broker
- Data flows and data flow graphs
- Akri connectors
- Device inbound endpoints
- Assets

For distributed resources, such as data flows and assets, the system aggregates health from multiple instances or subcomponents to provide a single, meaningful status.

### Staleness and freshness

To ensure health data remains trustworthy:

- Components periodically refresh their health status, even when no changes occur.
- If a resource doesn't report health within 15 minutes, its status automatically becomes **Unknown**.

This approach prevents stale information from being misinterpreted as healthy.

### Diagnostic details

When a resource is **Degraded** or **Unavailable**, you can access additional information to help you troubleshoot:

- [**Reason code**](#appendix-reason-codes-for-health-status) – a stable, documented identifier describing the failure type.
- **Message** – a human-readable explanation.
- **Timestamps** – when the issue started and when the status was last updated.

In the operations experience and the Azure portal, you can filter and group resources by health state and drill into the details for faster investigation.

## Metrics (historical behavior)

> [!NOTE]
> Before you can view metrics, you must deploy the observability stack. For setup instructions, see [Deploy observability resources](howto-configure-observability.md).

While health status shows the current state, metrics provide historical insight into how your system behaves over time.

Azure IoT Operations uses an open, standards-based observability pipeline built on:

- **OpenTelemetry Collector** - Deployed to the cluster to collect and export metrics from Azure IoT Operations components.
- **Azure Monitor managed service for Prometheus** - A fully managed Prometheus-compatible monitoring service that stores and queries metrics in the cloud.
- **Azure Managed Grafana** - A unified dashboard experience for visualizing health, metrics, and logs.

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

Azure IoT Operations provides a [single, unified Grafana dashboard](https://github.com/Azure-Samples/explore-iot-operations/tree/main/samples/observability/grafana-dashboard) that brings health, metrics, and logs together.

Key characteristics of the dashboard include:

- A health overview, visible at the top.
- Component-specific sections that load only when expanded.
- Metrics and logs shown side-by-side for common troubleshooting workflows.

This design supports the majority of monitoring scenarios without requiring you to jump between tools. For information about available metrics, see the [Available metrics](../reference/observability-metrics-mqtt-broker.md) section in the table of contents.

## Metrics documentation and customization

In addition to built-in dashboards, Azure IoT Operations provides documentation for all exposed metrics. This documentation helps you:

- Understand what each metric represents.
- Build custom dashboards tailored to your environment.
- Extend monitoring to optional components and connectors as needed.

## Onboard observability

Azure IoT Operations simplifies observability setup by using a single command to configure the required Azure and edge-side resources. For detailed instructions, see [Deploy observability resources](howto-configure-observability.md).

At a high level, enabling observability:

- Creates or validates Azure Monitor, Grafana, and Log Analytics resources.
- Configures the Arc-connected cluster without requiring direct Kubernetes access.
- Deploys and configures the OpenTelemetry collector.
- Wires Azure IoT Operations components to emit metrics automatically.

This approach reduces setup complexity and ensures consistent defaults.

## How health status and metrics work together

Health status and metrics are complementary signals:

| Aspect | Health status | Metrics |
|--------|---------------|---------|
| Purpose | Current state snapshot | Historical trends and patterns |
| Visibility | Operations experience and Azure portal | Grafana dashboards |
| Use case | "Is my system healthy right now?" | "What happened over the last hour or day?" |

### Example

If a data flow target becomes unreachable:

- **Metrics** show error counts increasing and throughput dropping.
- **Health status** changes to **Degraded** or **Unavailable** with a [reason code](#appendix-reason-codes-for-health-status).

After recovery:

- **Health status** returns to **Available**.
- **Metrics** preserve the historical record of the incident.

Together, these signals help you detect issues quickly and understand their impact.

## Appendix: reason codes for health status

When a resource reports **Degraded** or **Unavailable**, it includes a reason code that identifies the underlying issue. This feature enables faster troubleshooting without needing to immediately dive into logs. The following list shows the possible reason codes with detailed explanations and suggested action items.

> [!NOTE]
> In the following descriptions, placeholders such as `{error}`, `{e}`, or `{err}` represent the actual error message returned at runtime.

### Data flows

| Reason code | Description | Recommended action |
|------------|-------------|--------------------|
| `DataflowTransformSourceSchemaRetrievalFailed` | Failed to retrieve the source schema for a transform. | Verify schema reference and schema registry connectivity. |
| `DataflowTransformTargetSchemaRetrievalFailed` | Failed to retrieve the target schema for a transform. | Verify schema reference and schema registry connectivity. |
| `DataflowTransformConfigurationFailed` | Failed to build the transform pipeline. | Review the data flow transform configuration. |
| `DataflowTransformEnrichDataFailed` | Failed to enrich data during transform processing. | Check Broker state store connectivity and dataset configuration. |
| `DataflowTransformSourceChannelClosed` | Source input channel closed unexpectedly. | Restart the data flow pipeline if the issue persists. |
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

### Assets, devices, and connectors

| Reason code | Description | Suggested action |
|------------|-------------|------------------|
| `RestConnectorHttpClientCreationFailure` | Failed to create REST client: `{error}`. | Check TLS configuration and authentication settings. |
| `RestConnectorHttpRequestFailure` | HTTP request failed after retries: `{error}`. | Verify endpoint availability and network connectivity. |
| `RestConnectorSchemaGenerationFailure` | Failed to create message schema. Response data might be malformed or in an unexpected format. | Validate the response payload format and schema expectations. |
| `RestConnectorWasmProcessingFailure` | WASM graph processing failed: `{err}`. | Check WASM module configuration and input data format. |
| `SseConnectorSchemaGenerationFailure` | Failed to create message schema. Event data might be malformed or in an unexpected format. | Validate the event payload and schema configuration. |
| `SseConnectorStreamClosed` | SSE client stream closed unexpectedly. | Verify endpoint stability and restart the connector if needed. |
| `SseConnectorStreamError` | SSE stream error: `{e}`. | Check endpoint availability and network connectivity. |
| `MqttConnectorActionNotReady` | Management action is not ready: `{reason}`. | Verify that the management action prerequisites are satisfied. |
| `MqttConnectorPublishFailed` | PUBACK indicated failure: `{e}`. | Check broker connectivity, topic configuration, and permissions. |
| `MqttConnectorPublishFailed` | Failed to complete publish: `{e}`. | Verify MQTT broker availability and authentication settings. |
| `MqttConnectorPublishFailed` | Failed to publish: `{e}`. | Validate topic configuration and broker health. |
| `MqttConnectorSchemaCreationFailed` | Failed to create message schema, likely due to malformed data. | Inspect input payload and schema definitions. |
| `MqttConnectorSchemaReportingFailed` | Failed to report message schema, likely due to malformed data. | Validate schema reporting configuration and payload format. |
| `MqttConnectorWasmGraphCreationFailed` | Failed to create or load WASM graph for data transformation. | Verify WASM graph configuration and availability. |
| `MqttConnectorWasmGraphUpdateFailed` | Failed to update WASM graph for data transformation. | Check WASM graph update process and dependencies. |
| `MqttConnectorWasmProcessingFailed` | WASM graph failed to process message: `{err}`. | Inspect WASM module logs and input data. |
| `MqttConnectorWasmSchemaCreationFailed` | Failed to create message schema from WASM-processed data. | Validate WASM output and schema expectations. |

### Broker

| Reason code | Description | Suggested action |
|------------|-------------|------------------|
| `BrokerReplicaFailed` | Broker is in a failed state. | Collect and review the support bundle. |
| `BrokerPartitionFailed` | At least one backend replica is down. | Check broker replica health and restart failed replicas if necessary. |
| `BrokerProbeFailed` | Probe failed: Failed Operation CONNECT (1/8). | Verify broker connectivity, configuration, and network conditions. |

## Next steps

- [Configure observability for your Azure IoT Operations deployment](howto-configure-observability.md)
- [Clean up observability resources](howto-clean-up-observability-resources.md)
