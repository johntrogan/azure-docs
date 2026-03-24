---
title: Azure IoT Operations networking
description: Learn about Azure IoT Operations networking
author: dominicbetts
ms.subservice: layered-network-management
ms.author: dobett
ms.topic: concept-article
ms.custom:
  - ignite-2023
ms.date: 07/08/2025

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

## Layered networking sample

In industries like manufacturing, segmented networking architectures (such as the [Purdue Network Architecture](https://en.wikipedia.org/wiki/Purdue_Enterprise_Reference_Architecture)) are common. These architectures create layers that minimize or block lower-level segments from connecting to the internet. Azure IoT Operations supports secure management of devices in these layered networks using open, industry-recognized software, and Kubernetes-based configuration.

A [layered networking guidance sample](https://github.com/Azure-Samples/explore-iot-operations/tree/main/samples/layered-networking) is available in the Azure IoT Operations samples repository. The guidance describes the environment Microsoft uses to validate Azure IoT Operations deployments in a layered network. The sample and guidance show how to:

- Use Kubernetes-based configuration and networking primitives for layered environments.
- Connect devices in isolated networks at scale to [Azure Arc](/azure/azure-arc/) for application lifecycle management and remote configuration.
- Enforce security and governance across network levels with URL/IP allowlists and connection auditing.
- Ensure compatibility with all Azure IoT Operations services.

## Choose your networking approach

Before deploying, determine which networking approach fits your scenario:

- **Private-only with Arc Gateway:** If you have a single cluster that needs private connectivity to Azure without network segmentation between layers, use this approach. Arc Gateway consolidates Azure endpoints, Azure Firewall Explicit Proxy keeps traffic on private networks, and Private Link eliminates public endpoint exposure for data-plane services. See [Deploy Azure IoT Operations with private connectivity using Arc Gateway](howto-private-connectivity.md).
- **Layered network:** If you have a Purdue/ISA-95 segmented topology with multiple network layers (L2/L3/L4) and adjacent-only communication, use a layered network deployment. This approach adds Envoy proxy chaining, CoreDNS at each layer, and multi-cluster Azure IoT Operations deployments across layers. **If you have a layered topology, this is the recommended approach.** See [Deploy Azure IoT Operations in a layered network with private connectivity](howto-layered-network-private-connectivity.md).
- **Sovereign cloud:** If you operate in a regulated industry or region that requires data residency and compliance controls, consider deploying in an Azure sovereign cloud (for example, Azure Government, Azure China). This provides physical isolation and compliance certifications. However, it may require additional configuration and has a different set of available services.
