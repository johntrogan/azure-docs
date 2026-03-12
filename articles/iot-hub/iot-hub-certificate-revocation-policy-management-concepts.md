---
title: Certificate Revocation and Policy Management (Preview)
titleSuffix: Azure IoT Hub
description: This article discusses the concepts of revoking leaf certificates, revoking policies, deleting policies, and deleting credential resources in Azure IoT Hub certificate management.
author: cwatson-cat
ms.author: cwatson
ms.service: azure-iot-hub
services: iot-hub
ms.topic: conceptual
ai-usage: ai-generated
ms.date: 03/12/2026
#Customer intent: As an IoT Hub administrator or security team member, I want to understand the impact of revoking certificates and deleting certificate policies and credentials, so that I can make informed decisions about certificate lifecycle management.
---

# Certificate revocation and policy management concepts (preview)

Certificate management in Azure IoT Hub enables you to issue, manage, and retire X.509 certificates throughout their lifecycle. This article introduces the key concepts related to revoking leaf certificates, revoking policies, deleting policies, and removing credential resources. These operations are part of a coordinated lifecycle management strategy that helps you maintain your security posture when certificates expire, devices are decommissioned, or business requirements change.

[!INCLUDE [public-preview-banner](includes/public-preview-banner.md)]

## Relationship between certificate lifecycle operations

Certificate lifecycle in Azure Device Registry (ADR) follows a structured hierarchy: credentials (root CAs) → policies (issuing CAs) → leaf certificates (device certificates). Revocation and deletion operations cascade through this hierarchy based on your security and operational needs:

- **Revoke a leaf certificate** when a specific device is no longer trusted or its certificate is compromised.
- **Revoke a policy** when all certificates issued by that policy should be invalidated (for example, if the issuing CA itself is compromised).
- **Delete a policy** to remove the issuing CA after you handle existing certificates through revocation.
- **Delete a credential resource** to fully retire the root CA when it's no longer needed for any operations.

## Prerequisites

To perform certificate revocation and policy management operations, you must have the **Azure Device Registry Credentials Contributor** role assigned on the Azure Device Registry namespace. This role grants full access to manage credentials and policies (`microsoft.deviceregistry/namespaces/credentials/*` and `microsoft.deviceregistry/namespaces/credentials/policies/*`). 

If you don't have this role assigned, contact your Azure administrator to request the necessary permissions.

## Revoking a leaf certificate

When you revoke a leaf certificate, you invalidate a specific device's operational certificate, preventing that device from authenticating with IoT Hub by using that certificate. A revoked certificate becomes nonfunctional immediately. Revoke a leaf certificate when you need to invalidate a single device's access without affecting others. For example, revoke a leaf certificate when a device is suspected of being compromised, is being decommissioned, had its certificate exposed or leaked, or lost custody. To revoke a leaf certificate, select the device in Azure Device Registry and choose the **Revoke device certificates** action. You can revoke a certificate to force reprovisioning, or you can revoke and disable the device to permanently disconnect it.

### Impact of revoking a leaf certificate

Revoking a leaf certificate immediately affects the device that uses it but doesn't affect anything else in your certificate hierarchy.

- **Device authentication blocked**: The device can't authenticate with IoT Hub by using the revoked certificate. The device is effectively disconnected.
- **Device must re-provision**: To resume operation, the device must go through the provisioning process again with Device Provisioning Service (DPS) to get a new certificate.
- **No impact on policy or other certificates**: Revoking a single leaf certificate doesn't affect other devices or the policy that issued it. Other certificates issued by the same policy stay valid.
- **Audit trail**: The revocation event is recorded, so you can track when and why the certificate was invalidated.

## Revoking a policy

Revoking a policy invalidates the entire issuing certificate authority (ICA) that issues device certificates. This operation affects all leaf certificates that the policy issued, marking them as revoked. Revoke a policy when the risk or change applies to an entire group of devices rather than a single device. For example, revoke a policy when you suspect the issuing CA's private key is compromised, when you want to force a full re-provisioning cycle, or when compliance requirements mandate a complete refresh of a certificate chain. To revoke a policy, find the policy in the Azure portal and select **Revoke certificates**.

### Impact of revoking a policy

Because a policy is an issuing CA, revoking it affects every device whose certificate it issued.

- **All issued certificates revoked**: Every leaf certificate the policy issued becomes invalid immediately. All devices holding certificates from that policy lose the ability to authenticate.
- **Policy remains in place**: The policy isn't deleted but is marked as revoked. You can still query its history and audit information.
- **Devices must re-provision**: All affected devices must go through the provisioning process again with a different policy (if one exists) to get new certificates.
- **Other policies unaffected**: If you have multiple policies in your ADR namespace, revoking one policy doesn't affect certificates issued by other policies.
- **Security containment**: This operation is useful when you detect that a policy (or the private key used by the issuing CA) is compromised or when you want to perform a complete refresh of device certificates.
- **Microsoft-managed Root CA**: Azure PKI infrastructure handles the revocation. Microsoft maintains and manages the revocation list.
- **External Root CA**: If an external root CA signs your policy, you must ensure that the revocation also propagates to that external CA's certificate revocation list (CRL) or OCSP responder.

## Deleting a policy

Deleting a policy removes the issuing certificate authority (ICA) after all its certificates are revoked or expired. This operation is a final, permanent cleanup step. Unlike revocation, deletion removes the policy configuration entirely rather than marking it as invalid. Delete a policy only after you confirm that all certificates it issued are revoked or expired and no active devices depend on it. To delete a policy, find the policy in the Azure portal and select **Delete**.

### Impact of deleting a policy

Deleting a policy permanently removes the policy configuration. Unlike revocation, deletion doesn't just mark the policy as invalid.

- **Policy is permanently removed**: The policy and its configuration are deleted from your ADR namespace. This operation can't be undone.
- **Dependent resources**: Any device enrollment linked to this policy in Device Provisioning Service (DPS) can't issue new certificates. DPS enrollments referencing the deleted policy must be updated to use a different policy or disabled.
- **Audit information**: While the policy is deleted, you might retain historical records and audit logs related to the policy based on your compliance requirements.
- **No impact on active certificates**: If certificates issued by this policy are still valid, deleting the policy doesn't immediately invalidate them. However, new certificate issuance through this policy isn't possible.

## Deleting a credential resource

Deleting a credential resource removes the root certificate authority (root CA) from your ADR namespace. This operation removes the root of trust for the entire hierarchy. Delete a credential resource only when you're certain no downstream policies or devices need it, such as when the root CA reaches end-of-life, you're sunsetting a complete certificate hierarchy, or compliance policies require removal of old certificate infrastructure. To delete a credential resource, go to Azure Device Registry, select the namespace, and under **Credential policies**, select the credential resource and **Delete**.

### Impact of deleting a credential resource

Deleting a credential removes the root of trust for an entire ADR namespace's certificate hierarchy.

- **Root CA is permanently removed**: The credential (root CA) and all its associated metadata are deleted from your ADR namespace.
- **Cascading deletion**: If any policies are still associated with this credential, you must first revoke or delete those policies before you can delete the credential.
- **IoT Hub synchronization**: The root CA is removed from the synchronized certificate lists on linked IoT Hubs. IoT Hub no longer trusts certificates in this chain.
- **No new certificate issuance**: New certificate issuance through any policy signed by this credential isn't possible.
- **Irreversible operation**: Deleting a credential can't be undone. To use certificate management with this chain again, you must create a new credential and policies.

## Certificate lifecycle best practices

Revoking certificates and deleting policies are high-impact operations that are difficult or impossible to reverse. A well-defined lifecycle strategy helps you avoid unplanned device outages, reduce your security exposure, and respond quickly when incidents occur. The following practices apply whether you're managing a handful of devices or a fleet of thousands.

- **Monitor certificate expiration**: Actively track when certificates are approaching expiration and plan renewal schedules to avoid service interruptions.
- **Use separate policies for device cohorts**: Create policies for different groups of devices (for example, by manufacturer or device model). This approach limits the impact if you need to revoke a policy.
- **Maintain audit records**: Use Azure audit logs to track all certificate revocation, policy deletion, and credential deletion operations for compliance and security investigations.

## Related content

- [Key concepts for certificate management](iot-hub-certificate-management-concepts.md)
- [What is certificate management (preview)?](iot-hub-certificate-management-overview.md)
- [Get started with ADR and certificate management in IoT Hub](iot-hub-device-registry-setup.md)
- [Authenticate devices with X.509 CA certificates](authenticate-authorize-x509.md)
- [Azure role-based access control (RBAC) for IoT Hub](authenticate-authorize-azure-ad.md)
