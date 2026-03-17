---
title: What is Microsoft-backed X.509 Certificate Management (Preview)?
titleSuffix: Azure IoT Hub
description: This article discusses the basic concepts of how certificate management in Azure IoT Hub helps users manage device certificates.
author: cwatson-cat
ms.author: cwatson
ms.service: azure-iot-hub
services: iot-hub
ms.topic: overview
ms.date: 03/16/2026
ai-usage: ai-assisted
#Customer intent: As a developer new to IoT, I want to understand what certificate management is and how it can help me manage my IoT device certificates.
---

# What is Microsoft-backed X.509 certificate management (preview)?

Certificate management is an optional feature of Azure Device Registry (ADR) that you can use to issue and manage X.509 certificates for your IoT devices. It configures a dedicated, cloud-based public key infrastructure (PKI) for each ADR namespace, without requiring on-premises servers, connectors, or hardware. It manages certificate issuance and renewal for IoT devices that are provisioned to that ADR namespace. Devices use these X.509 certificates to authenticate with IoT Hub.

To use certificate management, you also need to use IoT Hub, [Azure Device Registry (ADR)](iot-hub-device-registry-setup.md), and [Device Provisioning Service (DPS)](../iot-dps/index.yml). Certificate management is currently in public preview.

[!INCLUDE [iot-hub-public-preview-banner](includes/public-preview-banner.md)]

## Overview of features

The following features are supported with certificate management for IoT Hub devices in preview:

| Feature | Description |
| --- | --- |
| Create multiple certificate authorities (CA) in an ADR namespace | Create two-tier PKI hierarchy with root and issuing CA in the cloud. |
| Create a unique root certificate authority (CA) per ADR namespace | Create up to one root CA, also known as a credential, in your ADR namespace. |
| Create up to one issuing CA per policy | Create up to one issuing CA, also known as a policy, in your ADR namespace and customize validity periods for issued certificates. |
| Signing and encryption algorithms | Certificate management supports ECC (ECDSA) and the NIST P-384 curve. |
| Hash algorithms | Certificate management supports SHA-384. |
| HSM keys (signing and encryption) | Keys are provisioned by using [Azure Managed Hardware Security Module (Azure Managed HSM)](/azure/key-vault/managed-hsm/overview). CAs you create within your ADR namespace automatically use HSM signing and encryption keys. No Azure subscription is required for Azure HSM. |
| End-entity certificate issuance and renewal | The issuing CA signs end-entity certificates, also known as leaf or device certificates, and delivers them to the device. The issuing CA can also renew leaf certificates. |
| At-scale provisioning of leaf certificates | Use policies you define in your ADR namespace to link directly to Device Provisioning Service enrollments during certificate provisioning. |
| Syncing of CA certificates with IoT Hubs | Sync policies you define in your ADR namespace to the appropriate IoT Hub so IoT Hub can trust devices that authenticate by using end-entity certificates. |

## Onboarding vs. operational credentials

Certificate management supports issuance and renewal for end-entity **operational certificates**. It doesn't manage onboarding credentials.

- **Onboarding credential:** A device uses this credential to authenticate with [Device Provisioning Service (DPS)](../iot-dps/index.yml) during provisioning. Supported onboarding credential types include X.509 certificates from a third-party certificate authority (CA), symmetric keys, and Trusted Platform Modules (TPM).
- **Operational certificate:** After the device provisions through DPS, Azure Device Registry (ADR) issues a short-lived X.509 certificate that the device uses to authenticate directly with IoT Hub.

For details, see [Certificate issuance in Azure IoT Hub certificate management](concept-certificate-issuance.md).

## How certificate management works

Certificate management uses IoT Hub, ADR, and DPS together to provide a managed public key infrastructure (PKI) experience for IoT devices.

At a high level:

- You create certificate management resources in ADR, including a namespace, credential, and policy.
- You configure DPS enrollments to use that ADR policy during provisioning.
- Devices provision through DPS and receive operational certificates signed by the policy's issuing CA.
- IoT Hub trusts those device certificates after ADR syncs the issuing CA certificate to linked hubs.

The following image shows the certificate hierarchy and service integration model.

:::image type="content" source="media/certificate-management/device-registry-certificate-management.png" alt-text="Diagram showing how Azure Device Registry integrates with IoT Hub and DPS for certificate management." lightbox="media/certificate-management/device-registry-certificate-management.png":::

For more information, see [Certificate issuance in Azure IoT Hub certificate management](concept-certificate-issuance.md).

## Certificate renewal

Certificate renewal uses the same workflow as first-time certificate issuance. When a device detects that its operational certificate is approaching expiration, it initiates another DPS registration call and submits a new certificate signing request (CSR).

Each device is responsible for monitoring its certificate validity period and triggering renewal before expiration to avoid connection interruptions.

For more information, see [Certificate renewal in Azure IoT Hub certificate management](concept-certificate-renewal.md).

## Disable a device

Certificate management doesn't support certificate revocation during public preview. To remove the connection of a device that uses an X.509 operational certificate, you can disable the device in the IoT Hub. To disable a device, see [Disable or delete a device](create-connect-device.md#disable-or-delete-a-device).

## Limits and quotas

See [Azure subscription and service limits](../azure-resource-manager/management/azure-subscription-service-limits.md#azure-iot-hub-limits) for the latest information about limits and quotas for certificate management with IoT Hub.

## Related content

- [FAQ: What is new in Azure IoT Hub?](iot-hub-faq.md)
- [Certificate issuance in Azure IoT Hub certificate management](concept-certificate-issuance.md)
- [Certificate renewal in Azure IoT Hub certificate management](concept-certificate-renewal.md)
- [Key concepts for certificate management](iot-hub-certificate-management-concepts.md)
- [Get started with ADR and certificate management in IoT Hub](iot-hub-device-registry-setup.md)
- [Integration with Azure Device Registry](iot-hub-device-registry-overview.md)
