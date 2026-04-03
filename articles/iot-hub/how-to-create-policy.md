---
title: Create or edit a policy with a Microsoft root CA in Azure Device Registry
titleSuffix: Azure IoT Hub
description: Create or edit a policy in your Azure Device Registry namespace to issue Microsoft-backed X.509 certificates for IoT devices.
author: cwatson-cat
ms.author: cwatson
ms.service: azure-iot-hub
services: iot-hub
ms.topic: how-to
ai-usage: ai-generated
ms.date: 03/16/2026
zone_pivot_groups: iot-hub-deployment-methods
#Customer intent: As an IoT administrator, I want to create or edit a policy in Azure Device Registry so I can issue Microsoft-backed X.509 device certificates with the validity period my deployment requires.
---

# Create or edit a policy with a Microsoft root CA (preview)

Create or edit a policy within your [Azure Device Registry (ADR)](iot-hub-device-registry-overview.md) namespace to manage an __Issuing CA__ that is signed by your namespace's unique __Root CA__.

Use this workflow if you want Azure Device Registry to provide a fully managed PKI for your namespace. When a device requests a certificate, the platform returns a full certificate chain consisting of:

__The Device Certificate:__ Unique to the specific IoT device.

__The Issuing CA (ICA):__ The CA managed by ADR that signs the device request.

- __The Namespace Root CA:__ The unique, namespace-level root managed by the credential resource.

This ensures that your device identities are cryptographically scoped to their namespace, providing high tenant isolation and a simplified management experience without the need for an external Private PKI.

[!INCLUDE [iot-hub-public-preview-banner](includes/public-preview-banner.md)]

In Certificate Management, a credential manages the namespace-level root CA, and a policy manages the issuing CA that signs device certificates. 

## Prerequisites

Before you begin, make sure you have:

- An active Azure subscription. If you don't have one, create a [free account](https://azure.microsoft.com/free/).
- An existing Azure Device Registry namespace. For setup steps, see [Deploy Azure IoT Hub with ADR integration](iot-hub-device-registry-setup.md).
- A configured credential in the ADR namespace. For setup steps, see [Configure a credential in Azure Device Registry](how-to-configure-credential.md).
- Permissions to manage policies in the ADR namespace, such as the [Azure Device Registry Credentials Contributor](../role-based-access-control/built-in-roles/internet-of-things.md#azure-device-registry-credentials-contributor) role.

## Choose a configuration method

You can create a policy by using the Azure portal, Azure CLI, or PowerShell.

In this preview workflow, use the Azure portal when you need to change the validity period for an existing policy.

| Configuration method | Description |
| --- | --- |
| Select **Azure portal** at the top of the page | Create a policy and edit its validity period in the portal. |
| Select **Azure CLI** at the top of the page | Create a policy and verify its settings by using preview CLI commands. |
| Select **PowerShell script** at the top of the page | Run Azure CLI commands from PowerShell to create a policy and verify its settings. |

:::zone pivot="portal"

## Create a policy

Create a policy in your ADR namespace so device certificates are issued with the validity period your deployment requires.

1. Sign in to the [Azure portal](https://portal.azure.com).
1. Open your **Azure Device Registry** namespace.
1. Select **Credential policies**.
1. Select **Create policy**.
1. In the policy type options, select **Default (Microsoft-issued Root CA)**.
1. Enter policy settings, including the validity period required for your deployment.
1. Select **Create**.
1. Refresh the **Credential policies** list if needed, and verify that the new policy appears.
1. Select the policy and **Sync**.

## Edit a policy

Edit an existing policy to update its validity period when security or operational requirements change.

1. In your ADR namespace, select **Credential policies**.
1. Select the policy that you want to edit.

1. On the policy details page, select **Edit**.
1. Change the **Validity period** value.
1. Select **Save**.
1. Refresh the page.
1. In **Essentials**, verify that the updated validity period appears.

:::zone-end

:::zone pivot="azure-cli"

## Azure CLI prerequisites

Prepare Azure CLI and authenticate to Azure so you can run the policy commands against the correct subscription and namespace.

- [Azure CLI](/cli/azure/install-azure-cli) installed on your machine.
- The `azure-iot` extension. Install it by running the following command:

  ```azurecli
  az extension add --name azure-iot
  ```

- Sign in to Azure by running `az login`.

## Set variables for Azure CLI

Define shared variables before running the commands.

```azurecli
RG_NAME="<resource-group>"
NS_NAME="<adr-namespace>"
POLICY_NAME="<policy-name>"
VALIDITY_DAYS="<validity-days>"
```

## Create the policy with Azure CLI

Run this command to create a policy that chains to your credential.

```azurecli
az iot adr ns policy create \
  --namespace "$NS_NAME" \
  -g "$RG_NAME" \
  --name "$POLICY_NAME" \
  --cert-validity-days "$VALIDITY_DAYS"
```

To set a custom certificate subject during creation, add the `--cert-subject` parameter.

## Verify the policy with Azure CLI

Run this command to view the policy details.

```azurecli
az iot adr ns policy show \
  --namespace "$NS_NAME" \
  -g "$RG_NAME" \
  --name "$POLICY_NAME"
```

Verify that the policy is returned and that the displayed properties match the values you created.

Editing an existing policy through Azure CLI preview commands for this workflow isn't currently documented. Use the Azure portal steps in this article to change the validity period for an existing policy.

:::zone-end

:::zone pivot="script"


## PowerShell prerequisites

Prepare PowerShell and Azure CLI prerequisites so you can run the policy workflow reliably from a scripted shell environment.

- PowerShell 7.0 or later.
- [Azure CLI](/cli/azure/install-azure-cli) installed on your machine.
- The `azure-iot` extension. Install it by running the following command:

  ```powershell
  az extension add --name azure-iot
  ```
- Sign in to Azure by running `az login`.

## Set variables for PowerShell

Define shared variables before running the commands.

```powershell
$ResourceGroupName = "<resource-group>"
$NamespaceName = "<adr-namespace>"
$PolicyName = "<policy-name>"
$ValidityDays = "<validity-days>"
```

## Create the policy from PowerShell

Run this Azure CLI command from PowerShell to create a policy chained to your credential.

```powershell
az iot adr ns policy create `
  --namespace $NamespaceName `
  -g $ResourceGroupName `
  --name $PolicyName `
  --cert-validity-days $ValidityDays
```

To set a custom certificate subject during creation, add the `--cert-subject` parameter.

## Verify the policy from PowerShell

Run this Azure CLI command from PowerShell to view the policy details.

```powershell
az iot adr ns policy show `
  --namespace $NamespaceName `
  -g $ResourceGroupName `
  --name $PolicyName
```

Verify that the policy is returned and that the displayed properties match the values you created.

Editing an existing policy through Azure CLI preview commands for this workflow isn't currently documented. Use the Azure portal steps in this article to change the validity period for an existing policy.

:::zone-end

## Related content

Use these related articles to set up ADR integration, configure credentials, manage policy lifecycle actions, and review core certificate management concepts.

- [Deploy Azure IoT Hub with ADR integration](iot-hub-device-registry-setup.md)
- [Configure a credential in Azure Device Registry](how-to-configure-credential.md)
- [Revoke certificates and delete policies in Azure Device Registry](how-to-revoke-certificate-delete-policy.md)
- [Key concepts for certificate management (preview)](iot-hub-certificate-management-concepts.md)
