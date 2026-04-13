---
title: Create or Edit a Policy with an External Root CA in Azure Device Registry
titleSuffix: Azure IoT Hub
description: Create or edit an external CA policy in Azure Device Registry so you can use your external CA to issue IoT device certificates.
author: cwatson-cat
ms.author: cwatson
ms.service: azure-iot-hub
services: iot-hub
ms.topic: how-to
ms.date: 04/16/2026
ai-usage: ai-generated
#Customer intent: As an IoT administrator, I want to create or edit an external CA policy in Azure Device Registry so I can issue and manage device certificates by using my external CA lifecycle.
---

# Create or edit a policy with an external root CA (preview)

This article explains how to create or edit a policy within your [Azure Device Registry (ADR)](iot-hub-device-registry-overview.md) namespace to manage an issuing CA that is signed by your organization's __external root CA__.

Use this workflow if your organization maintains a private Public Key Infrastructure (PKI) and requires all IoT devices to chain up to a common trusted root. When a device requests a certificate via ADR, the platform returns a full __certificate chain__ consisting of:

- __The device certificate:__ Unique to the specific IoT device.

- __The Microsoft issuing CA (ICA):__ The CA managed by ADR that signs the device request.

- __The external root CA:__ Your organization’s trusted root, which has signed the Microsoft ICA.

Any service that trusts your corporate root CA automatically trusts the certificates issued to your IoT devices by Azure.

[!INCLUDE [iot-hub-public-preview-banner](includes/public-preview-banner.md)]

In ADR certificate management, a credential is the namespace-level root CA resource, and a policy is the issuing CA policy that signs device certificates.

## Prerequisites

Before you begin, make sure you have the required setup and permissions so you can create, activate, and edit an external CA policy without deployment delays.

- An active Azure subscription. If you don't have one, create a [free account](https://azure.microsoft.com/free/).
- An existing ADR namespace. For setup steps, see [Deploy Azure IoT Hub with ADR integration](iot-hub-device-registry-setup.md).
- A configured credential in the ADR namespace. For setup steps, see [Configure a credential in Azure Device Registry](how-to-configure-credential.md).
- Permissions to manage policies in the ADR namespace, such as the [Azure Device Registry Credentials Contributor](../role-based-access-control/built-in-roles/internet-of-things.md#azure-device-registry-credentials-contributor) role.
- Access to your external CA workflow so you can sign the ADR-generated certificate signing request (CSR) and provide a full signed chain file.

## Requirements for your external root CA

To use an external root CA, your root CA must meet the following requirements:

| Property| Requirements|
| -------- | -------- |
|Key type and cryptography|The CA key type must be ECC (OID 1.2.840.10045.2.1). RSA is rejected. ECC P-256, P-384, P-521 are supported. Explicit EC parameters are rejected, only named curve encoding (OID) is allowed. Signature algorithm must be SHA-256 or higher with ECDSA (ecdsa-with-SHA256, SHA384, or SHA512). SHA-1 is rejected.|
|Path length|Path length for your root CA must be set to 1.|
| Subject| The subject on the signed certificate must match the subject from the original CSR (case-insensitive with whitespace normalization)|
|Extensions|BasicConstraints must be present, marked critical, with CA:TRUE. KeyUsage must include DigitalSignature, KeyCertSign, and CrlSign. If the EKU extension is present, it must include ClientAuth (OID 1.3.6.1.5.5.7.3.2). If EKU is absent entirely, that's acceptable (unconstrained CA per RFC 5280). Subject Key Identifier (SKI) must be present. X.509 Version must be Version 3.|
|Validity period|Certificate must not be expired. Certificate NotBefore must not be in the future. Must have at least 365 days of remaining validity.|

## Create a policy

Choose the workflow that matches how you operate your environment. In this example, only the Azure portal workflow shows how to edit the policy validity period. If you prefer automation, use the Azure CLI workflow.

# [Azure portal](#tab/portal)

## Create and activate an external CA policy

Create a policy that uses your external CA, and then activate it after you upload the signed certificate chain from your PKI.

1. Sign in to the [Azure portal](https://portal.azure.com).

1. Open your **Azure Device Registry** namespace.

1. In the sidebar menu, under **Namespace resources**, select **Certificate management**.

1. Select **Create Policy**.

    :::image type="content" source="media/how-to-create-policy-external-certificate/certificate-management.png" alt-text="Screenshot showing the certificate management pane.":::

1. In the **Basics** tab, complete the fields as follows:

    | Property | Value |    
    | -------- | ----- |
    | **Name** | Enter a unique name for your policy. The name must be between 3 and 50 alphanumeric characters and can include hyphens (`'-'`). |
    | **Validity period (days)** | Enter the number of days the issued certificates are valid. |
    | **Select a Root CA for certificates in this policy** | Select **Use an external root CA**. |

    :::image type="content" source="media/how-to-create-policy-external-certificate/add-policy-external.png" alt-text="Screenshot showing the Create policy dialog.":::

1. Select **Next**, then **Review + create**.

1. Refresh the **Policies** list if needed. Select the policy and confirm the policy status is **Pending Activation**.

1. Select the **Pending Activation** link, then select **Download certificate signing request (CSR) file**.

    :::image type="content" source="media/how-to-create-policy-external-certificate/download-certificate.png" alt-text="Screenshot showing the link to download the certificate signing request.":::

1. Sign the CSR by using your external CA and prepare the signed chain file. The signed certificate's subject must exactly match the subject of the original CSR.

1. In the policy details, upload the signed certificate chain.

1. Activate the policy.

1. Refresh the policy details and verify the policy status is active.

## Edit policy validity period

Update the validity period for an existing external CA policy when certificate lifetime requirements change.

1. In the sidebar menu of your ADR namespace, under **Namespace resources**, select **Certificate management**.

1. Select the policy that you want to edit.

1. On the **Overview** page, select **edit** next to **Validity period**.

1. Change the **Validity period** value.

1. Select **Save**.

# [Azure CLI](#tab/cli)

## Azure CLI prerequisites

Prepare Azure CLI and sign in so the external CA policy commands run against the correct subscription, resource group, and ADR namespace.

- [Azure CLI](/cli/azure/install-azure-cli) installed on your machine.
- The `azure-iot` extension. Install it by running:

  ```azurecli
  az extension add --name azure-iot
  ```

- Sign in to Azure by running `az login`.

## Set variables for Azure CLI

Define reusable variables before you run create, activation, and verification commands.

```azurecli
RG_NAME="<resource-group>"
NS_NAME="<adr-namespace>"
POLICY_NAME="<policy-name>"
```

## Create an external CA policy with Azure CLI

Run the following command to create a policy that uses your external CA.

```azurecli
az iot adr ns policy create \
  --namespace "$NS_NAME" \
  -g "$RG_NAME" \
  --name "$POLICY_NAME" \
  --enable-byor
```

After creation, the policy is pending activation until you provide a signed certificate chain.

Look for `certificateSigningRequest` in the console output. Copy the value into a blank *.pem* file and save it on your local machine.

## Activate the external CA policy with Azure CLI

After you sign the ADR-generated CSR by using your external CA, run the following command to upload the signed chain file and activate the policy.

```azurecli
az iot adr ns policy activate-byor \
  --ns "$NS_NAME" \
  -g "$RG_NAME" \
  --policy-name "$POLICY_NAME" \
  --certificate-chain-file "<path-to-signed-chain-file.pem>"
```

## Verify the policy with Azure CLI

Run the following command to confirm policy properties and current activation state.

```azurecli
az iot adr ns policy show \
  --namespace "$NS_NAME" \
  -g "$RG_NAME" \
  --name "$POLICY_NAME"
```

Editing the validity period for an existing external CA policy through Azure CLI preview commands isn't currently documented for this workflow. Use the Azure portal steps in this article.

---

## Related content

- [Deploy Azure IoT Hub with ADR integration](iot-hub-device-registry-setup.md)
- [Configure a credential in Azure Device Registry](how-to-configure-credential.md)
- [Revoke certificates and delete policies in Azure Device Registry](how-to-revoke-certificate-delete-policy.md)
- [Key concepts for certificate management (preview)](iot-hub-certificate-management-concepts.md)