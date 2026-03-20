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
ms.date: 03/20/2026
#Customer intent: As an IoT developer or administrator, I want to understand how certificate renewal works in Azure IoT Hub certificate management so I can keep my devices connected without interruption.
---

# Certificate renewal in Azure IoT Hub certificate management (preview)

Certificate renewal replaces an operational certificate that is close to expiration, or already expired, with a new certificate. Certificate management issues short-lived operational certificates. Each IoT device must track certificate expiration and start renewal before expiration. This article explains when to renew, the available renewal paths, how to track expiration, and what to do if a certificate expires.

[!INCLUDE [iot-hub-public-preview-banner](includes/public-preview-banner.md)]

## When devices need to renew

Certificate management issues operational certificates with a validity period that you set when you configure the policy in Azure Device Registry (ADR). The validity period ranges from 7 to 90 days.

Short-lived certificates improve security because they reduce exposure time if a certificate is compromised. Each device must start renewal before its current operational certificate expires to keep connectivity to IoT Hub.

Plan renewal before the expiration date. Add renewal logic to your device firmware or application so it sends a renewal request when the certificate reaches a set percentage of its validity period, such as 80 percent.

## How renewal works

You can renew an operational certificate in either of these ways:

- **Repeat certificate issuance through DPS**: The device uses its onboarding credential, starts a new Device Provisioning Service (DPS) registration, and submits a new certificate signing request (CSR). This path follows the same flow as initial certificate issuance.
- **Submit a new CSR to IoT Hub**: The device creates a new key pair and CSR, then sends the CSR to IoT Hub for renewal. IoT Hub handles the renewal request, works with certificate management to get a new operational certificate, and returns the renewed certificate to the device.

In both paths, the device must detect when renewal is needed, create a new key pair and CSR, replace the local certificate, and reconnect with the new certificate.

### Option 1: Repeat certificate issuance through DPS

If the device needs a new certificate, it can reprovision through DPS in these steps:

1. The device detects that its current certificate is approaching expiration.
1. The device initiates a new registration call to DPS with its onboarding credential.
1. As part of the registration request, the device sends a new certificate signing request (CSR) that includes a freshly generated public key.
1. DPS forwards the CSR to the policy's issuing CA in ADR.
1. The issuing CA validates and signs the CSR and issues a new operational certificate.
1. DPS returns the new certificate to the device.
1. The device uses the new certificate for all subsequent connections to IoT Hub.

The existing IoT Hub connection stays active until the device replaces the certificate and reconnects with the new one.

### Option 2: Submit a CSR directly to IoT Hub

Some solutions might renew without repeating the full DPS registration flow. In this model, the device sends a new CSR directly to IoT Hub. At a high level:

1. The device detects that its current certificate is approaching expiration.
1. The device creates a new key pair and CSR.
1. The device sends the CSR to IoT Hub for renewal.
1. IoT Hub works with certificate management to request a new operational certificate.
1. IoT Hub returns the renewed certificate to the device.
1. The device replaces the old certificate and reconnects with the new certificate.

For more information about initial issuance, see [Certificate issuance in Azure IoT Hub certificate management](concept-certificate-issuance.md).

## Track certificate expiration on devices

Each device must track expiration for its operational certificate. Certificate management doesn't send automatic renewal notifications to devices. The operational certificate includes `Valid from` and `Valid until` fields. The device can read these fields to determine when the certificate expires and when to start renewal.

Your device firmware or application code must:

1. Read the `Valid until` field from the certificate after each successful provisioning.
1. Calculate the time remaining before expiration.
1. Trigger a renewal request before the certificate expires, either by reprovisioning through DPS or by submitting a new CSR directly to IoT Hub.

## Reporting certificate expiration with device twins

During public preview, devices should use [device twin reported properties](iot-hub-devguide-device-twins.md#device-twins) to report the `Valid from` and `Valid until` dates of the current operational certificate. These reported properties support fleet-wide visibility, such as dashboards that show devices with certificates close to expiration.

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

If a device doesn't renew its certificate before expiration, it loses the ability to authenticate with IoT Hub. IoT Hub rejects connections from devices that present expired certificates.

To recover a device with an expired certificate:

1. Confirm that the device still has its onboarding credential. This credential is used for initial provisioning, such as a symmetric key or TPM.
1. Have the device initiate a new registration call to DPS.
1. DPS issues a fresh operational certificate through ADR by using the same CSR-based provisioning flow.
1. The device connects to IoT Hub with the new certificate.

If a device loses its onboarding credential and its operational certificate expires, you must reprovision the device with a new onboarding credential through your organization's device management workflow.

## Relationship to certificate revocation

Certificate renewal addresses expiration and is separate from certificate revocation. Revocation is a security action that invalidates a device certificate before expiration. After revocation, the device must obtain a new certificate again, typically through DPS reprovisioning and the same CSR-based issuance flow. For more information about revocation, see [Certificate revocation and policy management concepts](concepts-certificate-policy-management.md).

Certificate management doesn't support certificate revocation during public preview. To remove access for a device that uses an X.509 operational certificate, you can disable the device in IoT Hub. For more information, see [Disable or delete a device](create-connect-device.md#disable-or-delete-a-device).

## Related content

- [Certificate issuance in Azure IoT Hub certificate management](concept-certificate-issuance.md)
- [Certificate revocation and policy management concepts](concepts-certificate-policy-management.md)
- [Key concepts for certificate management](iot-hub-certificate-management-concepts.md)
- [What is certificate management (preview)?](iot-hub-certificate-management-overview.md)
- [Device twins in Azure IoT Hub](iot-hub-devguide-device-twins.md)
- [Get started with ADR and certificate management in IoT Hub](iot-hub-device-registry-setup.md)
