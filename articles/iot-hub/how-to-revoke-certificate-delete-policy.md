---
title: Revoke certificates and delete policies in Azure Device Registry
titleSuffix: Azure IoT Hub
description: Learn how to revoke device and policy certificates, and delete policies and credential resources in Azure Device Registry for IoT Hub.
author: cwatson-cat
ms.author: cwatson
ms.service: azure-iot-hub
services: iot-hub
ms.topic: how-to
ai-usage: ai-generated
ms.date: 03/13/2026
zone_pivot_groups: iot-hub-deployment-methods
#Customer intent: As an IoT Hub administrator, I want to revoke certificates and delete policies or credential resources so I can protect production devices and manage certificate lifecycle operations safely.
---

# Revoke certificates and delete policies (preview)

This article shows you how to run certificate lifecycle operations for Azure Device Registry (ADR) in Azure IoT Hub:

- Revoke certificates at the policy level.
- Revoke certificates at the device level.
- Delete a policy.
- Delete a credential resource.

Use these procedures when you need to respond to a security event, retire certificate resources, or clean up certificate paths in production.

[!INCLUDE [iot-hub-public-preview-banner](includes/public-preview-banner.md)]

## Prerequisites

Before you begin, make sure you have:

- An active Azure subscription. If you don't have one, create a [free account](https://azure.microsoft.com/free/).
- An existing production deployment with IoT Hub Gen2 linked to an ADR namespace. For setup steps, see [Deploy Azure IoT Hub with ADR integration and certificate management](iot-hub-device-registry-setup.md).
- A configured credential and policy in the ADR namespace. For setup steps, see [Configure a Root CA credential in Azure Device Registry](how-to-configure-credential.md).
- Device Provisioning Service (DPS) configured for devices that use operational certificate issuance and rotation.
- The [Azure Device Registry Credentials Contributor](../role-based-access-control/built-in-roles/internet-of-things.md#azure-device-registry-credentials-contributor) role on the ADR namespace.

## Choose a method

You can run these operations from the Azure portal, Azure CLI, or PowerShell.

| Method | Description |
| --- | --- |
| Select **Azure portal** at the top of the page | Use the portal experience for policy, credential, and device lifecycle operations. |
| Select **Azure CLI** at the top of the page | Use Azure CLI commands for repeatable operations and automation workflows. |
| Select **PowerShell script** at the top of the page | Use PowerShell with Azure CLI commands in script-based production workflows. |

:::zone pivot="portal"

## Revoke certificates for a policy

Use these steps to rotate a policy issuer when you need to invalidate certificates issued by that policy.

1. Sign in to the [Azure portal](https://portal.azure.com).
1. Open your **Azure Device Registry** namespace.
1. Select **Credential policies**.
1. Select the target policy.
1. Select **Revoke certificates**.
1. Confirm the operation.
1. Return to **Credential policies** and refresh.

For a standard or service-managed policy, Azure Device Registry rotates the issuing CA and syncs the replacement CA to linked hubs.
For a BYOR policy, Azure Device Registry creates a new CSR and sets the policy to pending activation. You must upload a new signed chain and activate the policy.

## Revoke certificates for a device

Use these steps to rotate one device certificate when you need to isolate risk to a single device.

1. In your ADR namespace, select **Devices**.
1. Select the target device.
1. Select **Revoke certificate**.
1. (Optional) Select **Also disable device after revoking** if you need to block device authentication.
1. Confirm the operation.

## Delete a policy

Use these steps to remove a policy when you no longer need it for certificate issuance.

1. Open **Credential policies** in your ADR namespace.
1. Select the policy you want to delete.
1. Select **Delete**.
1. Confirm the delete operation.


## Delete a credential resource

Use these steps to remove a credential resource when you need to retire that certificate path.

1. Open **Credential policies** in your ADR namespace.
1. Select the credential resource.
1. Select **Delete**.
1. Confirm the delete operation.

:::zone-end

:::zone pivot="azure-cli"

## Set variables (Azure CLI)

Define shared variables first so you can reuse the same values across all commands and reduce input errors.

```azurecli
RG_NAME="<resource-group>"
NS_NAME="<adr-namespace>"
HUB_NAME="<iot-hub-name>"
POLICY_NAME="<policy-name>"
DEVICE_ID="<device-id>"
```

## Revoke certificates for a standard or service-managed policy (Azure CLI)

Run this command to rotate a standard policy issuer and trigger the service-managed revoke flow.

```azurecli
az iot adr ns policy revoke-issuer \
  --ns "$NS_NAME" \
  -g "$RG_NAME" \
  --policy-name "$POLICY_NAME" \
  -y
```

## Revoke certificates for a BYOR policy (Azure CLI)

Use this flow to revoke a BYOR policy issuer and then reactivate trust with a newly signed certificate chain.

1. Revoke the policy issuer.

   ```azurecli
   az iot adr ns policy revoke-issuer \
     --ns "$NS_NAME" \
     -g "$RG_NAME" \
     --policy-name "$POLICY_NAME" \
     -y
   ```

1. Sign the new CSR with your CA and create a certificate chain file.
1. Activate BYOR with the new signed chain.

   ```azurecli
   az iot adr ns policy activate-byor \
     --ns "$NS_NAME" \
     -g "$RG_NAME" \
     --policy-name "$POLICY_NAME" \
     --certificate-chain-file "<path-to-chain-file.pem>"
   ```

1. Sync credentials to linked IoT Hubs.

   ```azurecli
   az iot adr ns credential sync --ns "$NS_NAME" -g "$RG_NAME"
   ```

1. Run these commands to verify policy and hub certificate state after the revoke operation.

   ```azurecli
   az iot adr ns policy show \
     --ns "$NS_NAME" \
     -g "$RG_NAME" \
     --policy-name "$POLICY_NAME"
   
   az iot hub certificate list --hub-name "$HUB_NAME" -g "$RG_NAME"
   ```

   Verify that policy state and hub certificates reflect the new issuer state.

## Revoke certificates for a device (Azure CLI)

Run these commands to rotate one device certificate, with an option to disable the device at the same time.

Revoke and keep device enabled:

```azurecli
az iot adr ns device revoke \
  -n "$DEVICE_ID" \
  --ns "$NS_NAME" \
  -g "$RG_NAME" \
  -y
```

Revoke and disable device:

```azurecli
az iot adr ns device revoke \
  -n "$DEVICE_ID" \
  --ns "$NS_NAME" \
  -g "$RG_NAME" \
  --disable \
  -y
```

Run these commands to verify the device state and hub identity state after device revoke.

```azurecli
az iot adr ns device show \
  -n "$DEVICE_ID" \
  --ns "$NS_NAME" \
  -g "$RG_NAME"

az iot hub device-identity show \
  -n "$HUB_NAME" \
  -g "$RG_NAME" \
  -d "$DEVICE_ID"
```

Verify that the device state matches your revoke mode and that hub identity certificate information updates.

## Delete a policy (Azure CLI)

Run this command to remove a policy that you no longer need in the namespace.

```azurecli
az iot adr ns policy delete \
  --ns "$NS_NAME" \
  -g "$RG_NAME" \
  --policy-name "$POLICY_NAME" \
  -y
```

Run this command to confirm the deleted policy no longer appears.

```azurecli
az iot adr ns policy list --ns "$NS_NAME" -g "$RG_NAME"
```

Verify that the deleted policy no longer appears.

## Delete a credential resource (Azure CLI)

Run this command to remove the credential resource when you retire that trust anchor path.

```azurecli
az iot adr ns credential delete \
  --ns "$NS_NAME" \
  -g "$RG_NAME" \
  --credential-name default \
  -y
```

Run this command to confirm the credential resource is no longer available.

```azurecli
az iot adr ns credential show \
  --ns "$NS_NAME" \
  -g "$RG_NAME" \
  --credential-name default
```

Verify that the credential resource is no longer available.

:::zone-end

:::zone pivot="script"

## Set variables (PowerShell)

Define shared variables first so each command targets the same resources and stays easy to review.

```powershell
$ResourceGroupName = "<resource-group>"
$NamespaceName = "<adr-namespace>"
$HubName = "<iot-hub-name>"
$PolicyName = "<policy-name>"
$DeviceId = "<device-id>"
```

## Revoke certificates for a standard or service-managed policy (PowerShell)

Run this command to rotate a standard policy issuer by using a PowerShell workflow.

```powershell
az iot adr ns policy revoke-issuer --ns $NamespaceName -g $ResourceGroupName --policy-name $PolicyName -y
```

## Revoke certificates for a BYOR policy (PowerShell)

Use this flow to revoke a BYOR issuer and restore trust after you upload a newly signed chain.

```powershell
az iot adr ns policy revoke-issuer --ns $NamespaceName -g $ResourceGroupName --policy-name $PolicyName -y
az iot adr ns policy activate-byor --ns $NamespaceName -g $ResourceGroupName --policy-name $PolicyName --certificate-chain-file "<path-to-chain-file.pem>"
az iot adr ns credential sync --ns $NamespaceName -g $ResourceGroupName
```

Run these commands to verify policy and hub certificate state after revoke.

```powershell
az iot adr ns policy show --ns $NamespaceName -g $ResourceGroupName --policy-name $PolicyName
az iot hub certificate list --hub-name $HubName -g $ResourceGroupName
```

## Revoke certificates for a device (PowerShell)

Run these commands to rotate one device certificate, with an option to disable the device.

Revoke and keep device enabled:

```powershell
az iot adr ns device revoke -n $DeviceId --ns $NamespaceName -g $ResourceGroupName -y
```

Revoke and disable device:

```powershell
az iot adr ns device revoke -n $DeviceId --ns $NamespaceName -g $ResourceGroupName --disable -y
```

Run these commands to verify the device and hub identity state after device revoke.

```powershell
az iot adr ns device show -n $DeviceId --ns $NamespaceName -g $ResourceGroupName
az iot hub device-identity show -n $HubName -g $ResourceGroupName -d $DeviceId
```

## Delete a policy (PowerShell)

Run this command to remove an unused policy from the namespace.

```powershell
az iot adr ns policy delete --ns $NamespaceName -g $ResourceGroupName --policy-name $PolicyName -y
```

Run this command to confirm the policy no longer appears in policy results.

```powershell
az iot adr ns policy list --ns $NamespaceName -g $ResourceGroupName
```

## Delete a credential resource (PowerShell)

Run this command to remove a credential resource when you no longer need that certificate path.

```powershell
az iot adr ns credential delete --ns $NamespaceName -g $ResourceGroupName --credential-name default -y
```

Run this command to confirm the credential resource is no longer available.

```powershell
az iot adr ns credential show --ns $NamespaceName -g $ResourceGroupName --credential-name default
```

:::zone-end

## Related content

Use these links to review concept guidance and setup prerequisites for related certificate management tasks.

- Review operation impact in [Certificate revocation and policy management concepts](concepts-certificate-policy-management.md).
- Review certificate hierarchy in [Key concepts for certificate management](iot-hub-certificate-management-concepts.md).
- Review setup requirements in [Deploy Azure IoT Hub with ADR integration and certificate management](iot-hub-device-registry-setup.md).
