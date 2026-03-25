---
title: Azure IoT Operations networking
description: Learn about Azure IoT Operations networking
author: dominicbetts
ms.subservice: layered-network-management
ms.author: dobett
ms.topic: concept-article
ms.custom:
  - ignite-2023
ms.date: 03/24/2026

#CustomerIntent: As an operator, I want understand how to use Azure IoT Operations networking to secure my devices.
ms.service: azure-iot-operations
---

# Azure IoT Operations networking

Networking is a foundational aspect of deploying and managing distributed systems, especially in hybrid and multicloud environments. In Azure IoT Operations, secure networking enables reliable connectivity between on-premises resources, edge devices, and Azure services. Proper network configuration is essential for communication, security, and scalability of IoT Operations and Kubernetes clusters. This article describes key networking options for IoT Operations.

## Azure Arc gateway

The Azure Arc gateway acts as a network proxy that lets you simplify network configuration requirements by reducing the number of Azure endpoints to allow through your firewall. By routing traffic through the gateway, you can simplify firewall rules and reduce the need for complex network changes. This approach is especially useful for securely connecting isolated or segmented environments to Azure Arc and Azure IoT Operations.

For more information, see [Simplify network configuration requirements with Azure Arc gateway (preview)](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking).

## Explicit proxy usage

Azure Firewall Explicit Proxy allows you to direct Azure Arc and IoT Operations traffic through a managed firewall, providing enhanced security and monitoring. This is useful for organizations that require all outbound traffic to be inspected or logged, and helps meet compliance requirements by controlling and auditing network flows to Azure.

For more information, see [Access Azure services over Azure Firewall Explicit Proxy (Public Preview)](/azure/azure-arc/azure-firewall-explicit-proxy).

## Choose your networking approach

Before deploying, determine which networking approach fits your scenario:

- **Private-only with Arc Gateway:** If you have a single cluster that needs private connectivity to Azure without network segmentation between layers, use this approach. Arc Gateway consolidates Azure endpoints, Azure Firewall Explicit Proxy keeps traffic on private networks, and Private Link eliminates public endpoint exposure for data-plane services. See [Deploy Azure IoT Operations with private connectivity using Arc Gateway](howto-private-connectivity.md).
- **Layered network:** If you have a Purdue/ISA-95 segmented topology with multiple network layers (L2/L3/L4) and adjacent-only communication, use a layered network deployment. This approach adds Envoy proxy chaining, CoreDNS at each layer, and multi-cluster Azure IoT Operations deployments across layers. **If you have a layered topology, this is the recommended approach.** See [Tutorial: Deploy Azure IoT Operations in a layered network with private connectivity](../end-to-end-tutorials/tutorial-layered-network-private-connectivity.md).


## Layered networking

In the basic architecture described in [Azure IoT Operations Architecture Overview](../overview-iot-operations.md#architecture-overview), all the Azure IoT Operations components are deployed to a single internet-connected cluster. In this type of environment, component-to-component and component-to-Azure connections are enabled by default.

However, in many industrial scenarios, computing units for different purposes are located in separate networks. For example:

- Assets and servers on the factory floor
- Data collecting and processing solutions in the data center
- Business logic applications with information workers

In industries like manufacturing, segmented networking architectures (such as the [Purdue Network Architecture](https://en.wikipedia.org/wiki/Purdue_Enterprise_Reference_Architecture)) are common. These architectures create layers that minimize or block lower-level segments from connecting to the internet. Azure IoT Operations supports secure management of devices in these layered networks using open, industry-recognized software, and Kubernetes-based configuration.

### Example of Azure IoT Operations in a layered network

The following diagram shows Azure IoT Operations deployed to multiple clusters in different network segments. In the Purdue Network architecture, level 4 is the enterprise network, level 3 is the operation and control layer, and level 2 is the controller system layer. Only level 4 has direct internet access, and the other levels can only communicate with their adjacent levels.

:::image type="content" source="media/layered-network-architecture.png" alt-text="Diagram that shows layered networking architecture for industrial layered networks.":::

In this example, Azure IoT Operations is deployed to levels 2 through 4. At levels 3 and 4, the Envoy Proxy is deployed, and levels 2 and 3 have Core DNS set up to resolve the approved URI to the parent cluster, which directs them to the parent Envoy Proxy. This setup redirects traffic from the lower layer to the parent layer. It lets you Arc-enable clusters and keep an Arc-enabled cluster running.

With extra configuration, you can use this technique to direct traffic east-west. This route lets Azure IoT Operations components send data to other components at upper levels and create data pipelines from the bottom layer to the cloud. In a multilayer network, you can deploy Azure IoT Operations components across layers based on your architecture and data flow needs. This example gives you general ideas about where to place individual components.

- Place the connector for OPC UA at the lower layer, closer to your assets and OPC UA servers.
- Transfer data toward the cloud through the MQTT broker components in each layer.
- Use the Data Flows component on nodes with enough compute resources, because it typically uses more compute.

### Layered networking sample walkthrough

In industries like manufacturing, segmented networking architectures (such as the [Purdue Network Architecture](https://en.wikipedia.org/wiki/Purdue_Enterprise_Reference_Architecture)) are common. These architectures create layers that minimize or block lower-level segments from connecting to the internet. Azure IoT Operations supports secure management of devices in these layered networks using open, industry-recognized software, and Kubernetes-based configuration.

A [layered networking guidance sample](https://github.com/Azure-Samples/explore-iot-operations/tree/main/samples/layered-networking) is available in the Azure IoT Operations samples repository. The guidance describes the environment Microsoft uses to validate Azure IoT Operations deployments in a layered network. The sample and guidance show how to:

- Use Kubernetes-based configuration and networking primitives for layered environments.
- Connect devices in isolated networks at scale to [Azure Arc](/azure/azure-arc/) for application lifecycle management and remote configuration.
- Enforce security and governance across network levels with URL/IP allowlists and connection auditing.
- Ensure compatibility with all Azure IoT Operations services.

> [!NOTE]
> The guidance doesn't recommend specific practices or provide production-ready implementation, configuration, or operations details. The guidance doesn't make recommendations about production networking architecture.

To learn more about how to prepare for a production-ready deployment of Azure IoT Operations, see the [Azure IoT Operations production checklist](../howto-production-checklist.md).    

To try the sample in a test environment, follow the step-by-step walkthrough:

1. [How Azure IoT Operations works in a layered network](https://github.com/Azure-Samples/explore-iot-operations/blob/main/samples/layered-networking/aio-layered-network.md)
2. [Configure the infrastructure](https://github.com/Azure-Samples/explore-iot-operations/blob/main/samples/layered-networking/configure-infrastructure.md)
3. [Arc-enable the K3s clusters](https://github.com/Azure-Samples/explore-iot-operations/blob/main/samples/layered-networking/arc-enable-clusters.md)
4. [Deploy Azure IoT Operations](https://github.com/Azure-Samples/explore-iot-operations/blob/main/samples/layered-networking/deploy-aio.md)
5. [Flow asset telemetry](https://github.com/Azure-Samples/explore-iot-operations/blob/main/samples/layered-networking/asset-telemetry.md)

> [!TIP]
> If you want to deploy Azure IoT Operations end-to-end in a layered network with private connectivity, see [Tutorial: Deploy Azure IoT Operations in a layered network with private connectivity](../end-to-end-tutorials/tutorial-layered-network-private-connectivity.md).