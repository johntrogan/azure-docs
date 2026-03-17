---
title: Certificate Renewal in Azure IoT Hub Certificate Management (Preview)
titleSuffix: Azure IoT Hub
description: Learn how certificate renewal works in Azure IoT Hub certificate management, including when to renew, how the renewal flow works, and how devices can track certificate expiration.
author: cwatson-cat
ms.author: cwatson
ms.service: azure-iot-hub
services: iot-hub
ms.topic: concept-article
ai-usage: ai-generated
ms.date: 03/16/2026
#Customer intent: As an IoT developer or administrator, I want to understand how certificate renewal works in Azure IoT Hub certificate management so I can keep my devices connected without interruption.
---

# Certificate renewal in Azure IoT Hub certificate management (preview)

Certificate renewal is the process of replacing an operational certificate that's expiring or has expired with a new one. Certificate management issues short-lived operational certificates, and each IoT device is responsible for monitoring its certificate's expiration date and initiating renewal before the certificate expires. This article explains when renewal is needed, how the renewal flow works, how devices can monitor certificate expiration, and what happens if a certificate expires.

[!INCLUDE [iot-hub-public-preview-banner](includes/public-preview-banner.md)]

## When devices need to renew

Certificate management issues operational certificates with a validity period that you set when you configure the policy in Azure Device Registry (ADR). The minimum validity period is 7 days and the maximum is 90 days.

Short-lived certificates improve your security posture by limiting the window of exposure if a certificate is compromised. Each device must initiate renewal before its current operational certificate expires to maintain uninterrupted connectivity to IoT Hub.

Plan to renew certificates before the expiration date, not at the expiration date. Implement renewal logic in your device firmware or application that triggers a renewal request when the certificate reaches a set percentage of its validity period, such as 80 percent.

## How renewal works

Certificate renewal uses the same flow as first-time certificate issuance. There's no separate renewal endpoint. When a device needs a new certificate, it reprovisions through Device Provisioning Service (DPS) by using the following steps:

1. The device detects that its current certificate is approaching expiration.
1. The device initiates a new registration call to DPS by using its onboarding credential.
1. As part of the registration request, the device sends a new certificate signing request (CSR) that includes a freshly generated public key.
1. DPS forwards the CSR to the policy's issuing CA in ADR.
1. The issuing CA validates and signs the CSR and issues a new operational certificate.
1. DPS returns the new certificate to the device.
1. The device uses the new certificate for all subsequent connections to IoT Hub.

The device's existing connection to IoT Hub remains active until the device replaces its certificate and reconnects by using the new one.

For more information about the first-time issuance flow, see [Certificate issuance in Azure IoT Hub certificate management](concept-certificate-issuance.md).

## Device responsibility for monitoring certificate expiration

Each device is responsible for tracking the expiration of its own operational certificate. Certificate management doesn't automatically send renewal notifications to devices. The operational certificate includes `Valid from` and `Valid until` fields that the device can read to determine when the certificate expires and when to initiate renewal.

Your device firmware or application code must:

1. Read the `Valid until` field from the certificate after each successful provisioning.
1. Calculate the time remaining before expiration.
1. Trigger a reprovisioning request to DPS before the certificate expires.

## Reporting certificate expiration with device twins

During public preview, devices should use [device twin reported properties](iot-hub-devguide-device-twins.md#device-twins) to report the `Valid from` and `Valid until` dates of their current operational certificate. These reported properties enable fleet-wide observability, such as building dashboards that surface devices with certificates approaching expiration.

The following example shows a reported properties structure:

```json
{
  "reported": {
    "certificate": {
      "validFrom": "2026-03-01T00:00:00Z",
      "validUntil": "2026-03-31T00:00:00Z"
    }
  }
}
```

After devices report these properties, you can use the [IoT Hub query language](iot-hub-devguide-query-language.md) to identify devices across your fleet that require renewal.

## What happens when a certificate expires

If a device doesn't renew its certificate before it expires, the device loses the ability to authenticate with IoT Hub. IoT Hub rejects connections from devices that present expired certificates.

To recover a device with an expired certificate:

1. Confirm the device still has its onboarding credential. This credential is used for initial provisioning, such as a symmetric key or TPM.
1. Have the device initiate a new registration call to DPS.
1. DPS issues a fresh operational certificate through ADR using the same CSR-based provisioning flow.
1. The device connects to IoT Hub by using the new certificate.

If a device loses its onboarding credential and its operational certificate has expired, you must re-provision the device with a new onboarding credential through your organization's device management workflow.

## Relationship to certificate revocation

Certificate renewal addresses expiration and is separate from certificate revocation. Revocation is a deliberate security action that invalidates a device's certificate before its expiration date. After revocation, the device must reprovision through DPS to receive a new certificate, following the same flow as renewal. For more information about revocation, see [Certificate revocation and policy management concepts](concepts-certificate-policy-management.md).

Certificate management doesn't support certificate revocation during public preview. To remove access for a device that uses an X.509 operational certificate, you can disable the device in IoT Hub. For more information, see [Disable or delete a device](create-connect-device.md#disable-or-delete-a-device).

## Related content

- [Certificate issuance in Azure IoT Hub certificate management](concept-certificate-issuance.md)
- [Certificate revocation and policy management concepts](concepts-certificate-policy-management.md)
- [Key concepts for certificate management](iot-hub-certificate-management-concepts.md)
- [What is certificate management (preview)?](iot-hub-certificate-management-overview.md)
- [Device twins in Azure IoT Hub](iot-hub-devguide-device-twins.md)
- [Get started with ADR and certificate management in IoT Hub](iot-hub-device-registry-setup.md)
