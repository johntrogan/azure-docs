---
title: Configure a credential in Azure Device Registry
titleSuffix: Azure IoT Hub
description: Configure a credential in your Azure Device Registry namespace to enable Microsoft-backed X.509 certificate management for IoT devices.
author: cwatson-cat
ms.author: cwatson
ms.service: azure-iot-hub
services: iot-hub
ms.topic: how-to
ai-usage: ai-generated
ms.date: 03/11/2026
zone_pivot_groups: iot-hub-deployment-methods
#Customer intent: As a developer setting up certificate management, I want to configure a root CA credential in my ADR namespace.
---

# Configure a credential in Azure Device Registry (preview)

When you enable [Microsoft-backed X.509 certificate management](iot-hub-certificate-management-overview.md) in your [Azure Device Registry (ADR)](iot-hub-device-registry-overview.md) namespace, you create single credential resource within that ADR namespace. A credential manages one unique root CA within your own cloud PKI.

[!INCLUDE [iot-hub-public-preview-banner](includes/public-preview-banner.md)]

Microsoft manages the PKI and root CA for your ADR namespace, so you don't need on-premises PKI infrastructure.

When you configure a credential, Microsoft:

- Generates and stores the root certificate in [Azure Managed HSM](/azure/key-vault/managed-hsm/overview)
- Manages the root certificate lifecycle
- Let's you create issuing CAs (policies) that the root CA signs

## Prerequisites

- An active Azure subscription. If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free/).
- An existing Azure Device Registry namespace. For setup instructions, see [Deploy Azure IoT Hub with ADR integration](iot-hub-device-registry-setup.md).
- Ensure that you have the privilege to perform role assignments within your target ADR namespace scope. Performing role assignments in Azure requires a [privileged role](../role-based-access-control/built-in-roles.md#privileged), such as Owner or User Access Administrator at the appropriate scope.

## Choose a configuration method

You can configure a root CA credential in your ADR namespace by using the Azure portal, Azure CLI, or PowerShell.

| Configuration method | Description |
|----------------------|-------------|
| Select **Azure portal** at the top of the page | Use the Azure portal to navigate to your ADR namespace and enable certificate management. |
| Select **Azure CLI** at the top of the page | Use the Azure CLI to enable certificate management and configure your root CA credential. |
| Select **PowerShell** at the top of the page | Use PowerShell to enable certificate management and configure your root CA credential. |

:::zone pivot="portal"

## Configure a credential by using the Azure portal

Follow these steps to configure your root CA credential.

1. Sign in to the [Azure portal](https://portal.azure.com).

1. Search for and select **Azure Device Registry** from the search bar.

1. Select your ADR namespace from the list.

1. In the namespace overview, select **Credential management**.

1. On the **Credential management** page, select **Enable credential management**.

1. A dialog appears asking you to confirm. Review the information and select **Enable**.


1. Azure provisions a root CA credential for your namespace. This process takes a few moments to complete.

1. After provisioning is complete, your root CA credential is ready to use. The credential is displayed on the **Credential management** page.

You can now create issuing CAs (policies) with a [Microsoft-issued certificate](how-to-create-policy.md) or with an [external CA](how-to-create-policy-external-certificate.md) within your namespace that are signed by your unique credential. Use these policies with Device Provisioning Service to issue and manage X.509 certificates for your IoT devices.

:::zone-end

:::zone pivot="azure-cli"

## Configure a credential by using the Azure CLI

Use the Azure CLI to enable credential management and configure a root CA credential in your ADR namespace.

### Prerequisites

- [Azure CLI](/cli/azure/install-azure-cli) installed on your machine
- The `azure-iot` extension. Install it by running:

   ```azurecli
   az extension add --name azure-iot
   ```

### Enable credential management

To enable credential management for your ADR namespace, run the following command:

```azurecli
az iot dps enrollment credential enable --adrs-endpoint <ADR_NAMESPACE_ENDPOINT> --adrs-resource-group <RESOURCE_GROUP_NAME>
```

Replace the following values:

- `<ADR_NAMESPACE_ENDPOINT>`: The endpoint URL of your ADR namespace (for example, `https://my-adr-namespace.api.iot.azure.com`)
- `<RESOURCE_GROUP_NAME>`: The name of the Azure resource group that contains your ADR namespace

### Verify credential configuration

After you enable credential management, verify that your root CA credential is provisioned by running:

```azurecli
az iot dps enrollment credential show --adrs-endpoint <ADR_NAMESPACE_ENDPOINT>
```

This command displays the details of your root CA credential, including its provisioning status and certificate information.

:::zone-end

:::zone pivot="script"

## Configure a credential using PowerShell

Use PowerShell to enable credential management and configure a root CA credential in your ADR namespace.

### Prerequisites

- [Azure PowerShell](/powershell/azure/install-azure-powershell) installed on your machine
- PowerShell 7.0 or higher
- The following modules installed:

   ```powershell
   Install-Module -Name Az.IoT -Force
   Install-Module -Name Az.KeyVault -Force
   ```

### Connect to Azure

Before running commands, authenticate to your Azure subscription:

```powershell
Connect-AzAccount
```

If you have multiple subscriptions, set the context to your target subscription:

```powershell
Set-AzContext -SubscriptionId <SUBSCRIPTION_ID>
```

Replace `<SUBSCRIPTION_ID>` with your Azure subscription ID.

### Enable credential management

To enable credential management for your ADR namespace, run the following command:

```powershell
$namespace = Get-AzResource -ResourceType "Microsoft.IoTOperationsMQ/namespaces" -ResourceGroupName "<RESOURCE_GROUP_NAME>" -Name "<NAMESPACE_NAME>"

$body = @{
    properties = @{
        credentialManagement = @{
            enabled = $true
        }
    }
} | ConvertTo-Json

Invoke-AzRestMethod -ResourceId "$($namespace.Id)" -Method PATCH -Payload $body
```

Replace the following values:

- `<RESOURCE_GROUP_NAME>`: The name of the Azure resource group that contains your ADR namespace
- `<NAMESPACE_NAME>`: The name of your ADR namespace

### Verify credential configuration

After you enable credential management, verify that your root CA credential is provisioned by running:

```powershell
$namespace = Get-AzResource -ResourceType "Microsoft.IoTOperationsMQ/namespaces" -ResourceGroupName "<RESOURCE_GROUP_NAME>" -Name "<NAMESPACE_NAME>"

$credential = Invoke-AzRestMethod -ResourceId "$($namespace.Id)/credentials" -Method GET

$credential.Content | ConvertFrom-Json | ConvertTo-Json
```

This command displays the details of your root CA credential, including its provisioning status and certificate information.

:::zone-end

## Next steps

After you configure your root CA credential, you can:

- [Create issuing CA policies](iot-hub-device-registry-overview.md) within your namespace to issue X.509 certificates for your IoT devices
- [Link an IoT Hub to your ADR namespace](iot-hub-device-registry-setup.md) to enable certificate-based authentication for your devices
- [Set up Device Provisioning Service](../iot-dps/quick-setup-auto-provision.md) enrollments to provision devices with issued certificates

For more information about certificate management and the complete workflow, see:

- [Azure Device Registry overview](iot-hub-device-registry-overview.md)
- [Microsoft-backed X.509 certificate management concepts](iot-hub-certificate-management-concepts.md)
- [Certificate management overview](iot-hub-certificate-management-overview.md)
