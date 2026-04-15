---
title: Best practices for namespaces and schema registries
titleSuffix: Azure Device Registry
description: Learn how to design Azure Device Registry namespaces and schema registries for your IoT solution, including when to create new instances and when to reuse existing ones.
author: dominicbetts
ms.author: dobett
ms.service: azure-iot
ms.topic: best-practice
ms.date: 04/15/2026
ai-usage: ai-assisted
#Customer intent: As an IoT solution architect, I want to understand how to design my namespace and schema registry structure so that I can organize devices and assets effectively and avoid rework later.
---

# Best practices for namespaces and schema registries in Azure Device Registry

[Azure Device Registry](../iot-operations/discover-manage-assets/overview-manage-assets.md) uses *namespaces* to organize devices and assets, and *schema registries* to manage message schemas. Because both resources act as long-lived organizational boundaries, it's important to plan them before you deploy.

This article helps you decide:

- When to create a new namespace or reuse an existing one.
- When to create a new schema registry or reuse an existing one.
- Which anti-patterns to avoid.

## Service applicability

Namespaces and schema registries are features of Azure Device Registry, which works with both Azure IoT Operations and Azure IoT Hub. The following table summarizes the current applicability:

| Feature | Azure IoT Operations | Azure IoT Hub |
|---|---|---|
| Namespaces | GA | Preview |
| Schema registries | GA | Not applicable |

> [!NOTE]
> Schema registries are currently used only in Azure IoT Operations scenarios, where data flows use schemas to describe, transform, and serialize messages at the edge. For more information, see [Understand message schemas](../iot-operations/connect-to-cloud/concept-schema-registry.md).

## Namespace planning

A namespace is a management and organizational boundary for devices and assets in Azure Device Registry. All devices and assets belong to exactly one namespace.

### Cardinality rules

- Each Azure IoT Operations instance or IoT Hub maps to exactly one namespace.
- Multiple Azure IoT Operations instances or IoT Hubs can share the same namespace.

Think of a namespace like an Azure resource group: a resource belongs to one resource group, but many resources can share a group. Customers typically group resources by organizational, product, or project boundaries rather than creating one resource group per resource. Apply the same principle to namespaces.

> [!IMPORTANT]
> Unlike resource groups, namespaces don't support moving resources. A device or asset's namespace assignment is permanent for its lifetime. You can recreate a resource in another namespace by using ARM or Bicep templates, but there's no direct move operation. Plan your namespace boundaries carefully before you deploy.

### When to create a new namespace

Create a new namespace when you need a distinct organizational or governance boundary, for example:

- **Separate operational sites.** A manufacturing company with factories in different cities creates one namespace per factory so that each site has its own device and asset scope.
- **Different business units.** An enterprise with independent divisions (energy, logistics, manufacturing) uses separate namespaces so that each division manages its own assets.
- **Access control isolation.** You need different role-based access control (RBAC) policies for different sets of devices. Because RBAC can be managed at the namespace level, separate namespaces give you a clear security boundary.
- **Service-level separation.** Certain platform capabilities, such as certificate management policies, are enabled or disabled at the namespace level. If you need different settings for different groups of devices, use separate namespaces.

### When to reuse an existing namespace

Reuse an existing namespace when the devices and assets logically belong together, for example:

- **Multiple Azure IoT Operations instances at the same site.** If you run two Azure IoT Operations clusters in the same factory, those clusters can share a single namespace. Devices and assets are visible across both instances.
- **Mixed connectivity patterns.** A site that uses both Azure IoT Operations (edge-connected) and IoT Hub (cloud-connected) can share a namespace so that all devices appear in a unified registry.
- **Scaling out within one business boundary.** Adding more devices or clusters within an existing organizational boundary doesn't require a new namespace.

### Namespace design examples

| Scenario | Recommended design | Rationale |
|---|---|---|
| Single factory, one Azure IoT Operations cluster | One namespace | All assets belong to the same site and team. |
| Single factory, two Azure IoT Operations clusters | One shared namespace | Both clusters serve the same site; sharing gives unified visibility. |
| Three factories in different regions | One namespace per factory | Each site has distinct teams, assets, and access control requirements. |
| Enterprise with separate divisions | One namespace per division | Each division manages its own devices and policies independently. |

## Schema registry planning

The schema registry, a feature of Azure Device Registry, is a synchronized repository in the cloud and at the edge. It stores definitions of messages coming from edge assets and exposes an API to access those schemas at the edge. For a detailed introduction, see [Understand message schemas](../iot-operations/connect-to-cloud/concept-schema-registry.md).

### When to create a new schema registry

Create a new schema registry only when the schemas are genuinely independent, for example:

- **Completely separate sites** with no shared asset types or message formats.
- **Different storage accounts** are required for schema storage at different sites.

### When to reuse an existing schema registry

In most cases, reuse an existing schema registry:

- **Multiple Azure IoT Operations instances at the same site** should share one schema registry. Message schemas logically originate from the assets and devices at a site, so there's no meaningful benefit to creating a separate registry for each instance.
- **Sites with overlapping asset types** benefit from sharing schemas rather than duplicating them across registries.

### Anti-patterns to avoid

| Anti-pattern | Why it's a problem | Recommended approach |
|---|---|---|
| One schema registry per Azure IoT Operations instance | Creates unnecessary duplication and management overhead. | Share one schema registry across all instances at a site. |
| Accepting deployment defaults without reviewing | Azure IoT Operations deployment scripts prompt you to create a new schema registry, which leads to proliferation. | During deployment, specify an existing schema registry if one already exists for your site. |

## Planning constraints and limits

The following limits affect namespace and schema registry design. For the full list, see [Azure Device Registry limits](/azure/azure-resource-manager/management/azure-subscription-service-limits#azure-device-registry-limits).

| Resource | Limit |
|---|---|
| Namespaces per subscription | 100 |
| Devices per namespace | 10,000 |
| Assets per namespace | 10,000 |
| Schema registries per subscription | 100 |

If your solution approaches these limits, consider whether your namespace design is too fine-grained (too many namespaces, each with few devices) or too coarse (one namespace that exceeds the device limit).

## Related content

- [What is asset and device management in Azure IoT Operations?](../iot-operations/discover-manage-assets/overview-manage-assets.md)
- [Understand message schemas](../iot-operations/connect-to-cloud/concept-schema-registry.md)
- [What is Azure IoT Hub?](../iot-hub/iot-concepts-and-iot-hub.md)
- [What is Azure IoT Operations?](../iot-operations/overview-iot-operations.md)
