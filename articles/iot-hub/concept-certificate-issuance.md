---
title: Certificate Issuance in Azure IoT Hub Certificate Management (Preview)
titleSuffix: Azure IoT Hub
description: Learn how Azure Device Registry issues X.509 certificates to IoT devices during provisioning, including the certificate hierarchy, issuance flow, and how IoT Hub trusts issued certificates.
author: cwatson-cat
ms.author: cwatson
ms.service: azure-iot-hub
services: iot-hub
ms.topic: concept-article
ai-usage: ai-generated
ms.date: 03/16/2026
#Customer intent: As an IoT developer or administrator, I want to understand how certificate issuance works in Azure IoT Hub certificate management so I can design a secure device provisioning workflow.
---

# Certificate issuance in Azure IoT Hub certificate management (preview)

Certificate issuance is the process by which Azure Device Registry (ADR) generates and delivers X.509 certificates to your Internet of Things (IoT) devices during provisioning. This article explains the certificate authority (CA) hierarchy that certificate management establishes, how ADR and Device Provisioning Service (DPS) work together to issue certificates at scale, and how IoT Hub trusts the issued certificates.

[!INCLUDE [iot-hub-public-preview-banner](includes/public-preview-banner.md)]

## Certificate authority hierarchy

Certificate management organizes certificate authorities in a two-tier hierarchy within each ADR namespace. IoT Hub uses this hierarchy to authenticate your devices through a chain of trust.

The hierarchy consists of three levels:

- **Root CA (credential):** The top-level, trust-anchor certificate authority for the namespace. Each ADR namespace contains exactly one root CA, called a *credential*. For a service-managed policy, Microsoft generates and maintains this root CA in [Azure Managed Hardware Security Module (HSM)](/azure/key-vault/managed-hsm/overview). For an external CA policy, your organization provides the root CA or an intermediate signing chain.

- **Issuing CA (policy):** An intermediate certificate authority that the root CA signs. The issuing CA lives within a *policy* resource in ADR and directly signs device certificates. Each credential supports one policy in the current preview. You configure the validity period that the issuing CA applies to all device certificates it issues, from a minimum of 7 days to a maximum of 90 days.

- **Device certificate (leaf certificate):** An end-entity certificate issued to a specific IoT device. The device uses this certificate to authenticate with IoT Hub over mutual transport layer security (TLS).

## How certificate issuance works

When a device provisions through DPS, ADR automatically issues an operational certificate to that device. You don't need to manually generate or deploy a certificate. The following steps describe the end-to-end issuance flow:

1. The IoT device connects to the DPS endpoint and authenticates by using its pre-configured onboarding credential, such as a symmetric key, X.509 certificate, or Trusted Platform Module (TPM). As part of this registration call, the device sends a certificate signing request (CSR) that includes the device's public key and its registration ID.
1. DPS assigns the IoT device to an IoT Hub based on the enrollment configuration.
1. The device identity is created in IoT Hub and registered to the ADR namespace.
1. DPS sends the CSR to the PKI. The PKI validates the request and forwards it to the policy linked to the DPS enrollment.
1. The policy's issuing CA signs and issues the operational certificate.
1. DPS returns the issued certificate and IoT Hub connection details to the device.
1. The device authenticates with IoT Hub by presenting its full certificate chain.

:::image type="content" source="media/certificate-management/operational-diagram.png" alt-text="Diagram that shows how Azure Device Registry integrates with IoT Hub and DPS for certificate management during provisioning." lightbox="media/certificate-management/operational-diagram.png":::

## Service-managed vs. external CA issuance

Certificate management supports two policy types for certificate issuance. The policy type you choose determines how the issuing CA is created and managed.

- **Service-managed policy:** Microsoft generates and maintains the issuing CA in Azure Managed HSM. You don't need to manage private keys, run signing operations, or maintain your own public key infrastructure (PKI). After you create the policy, ADR automatically syncs the issuing CA certificate to your linked IoT Hubs.

- **External CA policy:** You provide the root CA from your own private PKI. After you create the policy, ADR generates a CSR for the issuing CA. You sign that CSR by using your external CA, and then upload the signed certificate chain to activate the policy directly in ADR. You must also run a credential sync to push the issuing CA certificate to your linked IoT Hubs.

In either case, the provisioning experience for the device is the same: the device sends a CSR to DPS and receives a signed leaf certificate in return.

## Certificate signing request requirements

When a device provisions or reprovisions, it sends a CSR to DPS. DPS expects the CSR to meet the following requirements:

- **Format:** Base64-encoded distinguished encoding rules (DER) following the public key cryptography standards (PKCS) #10 specification. Privacy-enhanced mail (PEM) headers and footers can't be included.
- **Common name (CN):** The CN field must exactly match the device's DPS registration ID.
- **Key algorithm:** Elliptic curve (EC) key using the NIST P-384 curve. RSA keys aren't supported in the current preview.

For implementation examples, see [DPS device SDK samples](../iot-dps/libraries-sdks.md#device-sdks).

## Cryptographic algorithms

Certificate management uses the following cryptographic standards for all certificates issued by a service-managed policy:

| Property | Value |
|----------|-------|
| Key algorithm | ECC (ECDSA) |
| Curve | NIST P-384 (secp384r1) |
| Hash algorithm | SHA-384 |
| Key storage | Azure Managed HSM |

ECC with P-384 offers equivalent security to RSA at much smaller key sizes. This algorithm produces smaller certificates, faster TLS handshakes, and lower power consumption on constrained IoT devices.

## IoT Hub trust and credential sync

For a device to authenticate with IoT Hub by using its issued certificate, IoT Hub must trust the issuing CA that signed the device certificate. ADR manages this trust through credential sync, which pushes the issuing CA certificate from ADR to your linked IoT Hubs.

How sync works depends on the policy type:

- **Service-managed policy:** ADR syncs the issuing CA certificate to all linked IoT Hubs automatically when you create or revoke the policy.
- **External CA policy:** You must run credential sync manually after you activate the policy, and again after you revoke and reactivate the policy with a new certificate chain.

To run credential sync manually, use the following Azure CLI command:

```azurecli
az iot adr ns credential sync --namespace <namespace> -g <resource-group>
```

IoT Hub stores the issuing CA certificate and uses it to validate the certificate chain that your devices present during TLS authentication.

## Onboarding and operational certificate distinctions

Certificate management issues operational certificates, not onboarding credentials:

- **Onboarding credential:** The credential the device uses to authenticate with DPS during initial provisioning - for example, a symmetric key, X.509 certificate from a third-party CA, or TPM. Typically, you install onboarding credentials on the device before it ships and certificate management doesn't manage them.

- **Operational certificate:** The short-lived X.509 certificate ADR issues after the device provisions through DPS. The device uses the operational certificate to authenticate directly with IoT Hub during normal operation. Certificate management provisions, renews, and revokes operational certificates.

## Related content

- [Certificate renewal in Azure IoT Hub certificate management](concept-certificate-renewal.md)
- [Key concepts for certificate management](iot-hub-certificate-management-concepts.md)
- [What is certificate management (preview)?](iot-hub-certificate-management-overview.md)
- [Get started with ADR and certificate management in IoT Hub](iot-hub-device-registry-setup.md)
- [DPS device SDK samples](../iot-dps/libraries-sdks.md#device-sdks)
