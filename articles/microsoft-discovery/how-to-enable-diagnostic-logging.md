---
title: Enable diagnostic settings for Microsoft Discovery resources
description: Learn how to configure Azure Monitor diagnostic settings to collect platform logs from Microsoft Discovery workspaces, bookshelves, and supercomputers.
author: anzaman
ms.author: alzam
ms.service: azure
ms.topic: how-to
ms.date: 04/10/2026
---

# Enable diagnostic settings for Microsoft Discovery resources

When you have critical applications and business processes relying on Microsoft Discovery, you want to monitor those resources for availability, performance, and compliance. This article describes how to configure Azure Monitor diagnostic settings to collect platform logs from Microsoft Discovery resources.

Microsoft Discovery uses [Azure Monitor](/azure/azure-monitor/fundamentals/overview). If you're unfamiliar with the features of Azure Monitor common to all Azure services that use it, see [Monitor Azure resources with Azure Monitor](/azure/azure-monitor/essentials/monitor-azure-resource).

## Supported resource types and destinations

You can enable diagnostic settings on the following Microsoft Discovery resource types:

| Resource type | Resource provider path |
| --- | --- |
| Workspace | `Microsoft.Discovery/workspaces` |
| Bookshelf | `Microsoft.Discovery/bookshelves` |
| Supercomputer | `Microsoft.Discovery/supercomputers` |

Currently, the following destination is supported for diagnostic log export:

- **Azure Storage Account** — Archive logs for auditing, troubleshooting, and compliance.

## Prerequisites

- An Azure account with an active subscription. [Create an account for free](https://azure.microsoft.com/free/).
- The [Azure Az PowerShell module](/powershell/azure/install-azure-powershell) installed.
- **Monitoring Contributor** role or higher on the Discovery resource.
- **Storage Account Contributor** role or higher on the destination storage account.
- An [Azure Storage account](/azure/storage/common/storage-account-create) to receive the diagnostic logs.

## Connect to Azure

Sign in to Azure PowerShell and set the subscription context:

```azurepowershell
Connect-AzAccount
Set-AzContext -SubscriptionId "<subscription-id>"
```

## Configure the log category

Before you create a diagnostic setting, define which log categories to collect. You can enable all logs or only audit logs.

### [All logs](#tab/all-logs)

To collect all available log categories, run the following command:

```azurepowershell
$log = New-AzDiagnosticSettingLogSettingsObject `
    -Enabled $true `
    -CategoryGroup "allLogs"
```

### [Audit logs only](#tab/audit-logs)

To collect only audit logs, run the following command:

```azurepowershell
$log = New-AzDiagnosticSettingLogSettingsObject `
    -Enabled $true `
    -CategoryGroup "audit"
```

---

## Create a diagnostic setting

Use the `New-AzDiagnosticSetting` cmdlet to create a diagnostic setting for your Discovery resource. Replace the `<placeholder>` values with your resource information.

### [Workspace](#tab/workspace)

```azurepowershell
New-AzDiagnosticSetting `
    -Name "<diagnostic-setting-name>" `
    -ResourceId "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Discovery/workspaces/<workspace-name>" `
    -StorageAccountId "/subscriptions/<subscription-id>/resourceGroups/<storage-resource-group>/providers/Microsoft.Storage/storageAccounts/<storage-account-name>" `
    -Log $log
```

### [Bookshelf](#tab/bookshelf)

```azurepowershell
New-AzDiagnosticSetting `
    -Name "<diagnostic-setting-name>" `
    -ResourceId "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Discovery/bookshelves/<bookshelf-name>" `
    -StorageAccountId "/subscriptions/<subscription-id>/resourceGroups/<storage-resource-group>/providers/Microsoft.Storage/storageAccounts/<storage-account-name>" `
    -Log $log
```

### [Supercomputer](#tab/supercomputer)

```azurepowershell
New-AzDiagnosticSetting `
    -Name "<diagnostic-setting-name>" `
    -ResourceId "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Discovery/supercomputers/<supercomputer-name>" `
    -StorageAccountId "/subscriptions/<subscription-id>/resourceGroups/<storage-resource-group>/providers/Microsoft.Storage/storageAccounts/<storage-account-name>" `
    -Log $log
```

---

### Parameter reference

| Placeholder | Description |
| --- | --- |
| `<diagnostic-setting-name>` | A descriptive name for the diagnostic setting. |
| `<subscription-id>` | The Azure subscription ID that contains the Discovery resource. |
| `<resource-group>` | The resource group of the Discovery resource. |
| `<workspace-name>` | The name of the Microsoft Discovery workspace. |
| `<bookshelf-name>` | The name of the Microsoft Discovery bookshelf. |
| `<supercomputer-name>` | The name of the Microsoft Discovery supercomputer. |
| `<storage-resource-group>` | The resource group of the destination storage account. |
| `<storage-account-name>` | The name of the destination Azure Storage account. |

## Verify the diagnostic setting

To confirm that the diagnostic setting was created successfully, run the following command:

```azurepowershell
Get-AzDiagnosticSetting `
    -ResourceId "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Discovery/<resource-type>/<resource-name>"
```

Replace `<resource-type>` with `workspaces`, `bookshelves`, or `supercomputers`, and `<resource-name>` with the name of your resource.

A successful response returns the diagnostic setting name, the storage account destination, and the enabled log categories.

> [!NOTE]
> Diagnostic logs might take several minutes to appear in the storage account after you create the setting.

## Manage log retention

Logs exported to the storage account are stored in Azure Monitor resource-specific containers. To manage how long logs are retained, configure a [lifecycle management policy](/azure/storage/blobs/lifecycle-management-overview) on the destination storage account.

## Related content

- [Azure Monitor diagnostic settings](/azure/azure-monitor/essentials/diagnostic-settings)
- [Azure Monitor resource logs](/azure/azure-monitor/essentials/resource-logs)
- [Create a storage account](/azure/storage/common/storage-account-create)
- [Install Azure PowerShell](/powershell/azure/install-azure-powershell)
