---
title: Protect Azure NetApp Files volumes
description: Learn about how you can enable backup to protect your existing Azure NetApp Files volumes
services: azure-netapp-files
author: netapp-manishc
ms.service: azure-netapp-files
ms.topic: concept-article
ms.date: 10/22/2025
ms.author: anfdocs
---
# Protect volumes of Azure NetApp Files (preview)

You can protect your existing volume by enabling backup protection for the volume. This enhances data protection with an additional layer of protection without the need for manual setup.

## Register the feature 

Enable backup by default for Azure NetApp Files are currently in preview. You need to register the feature before using it for the first time. After registration, the feature is enabled and works in the background.

1.  Register the feature:

    ```azurepowershell-interactive
    Register-AzProviderFeature -ProviderNamespace Microsoft.NetApp -FeatureName ANFBackupByDefault
    ```

2. Check the status of the feature registration: 

    > [!NOTE]
    > The **RegistrationState** can remain in the `Registering` state for up to 60 minutes before changing to `Registered`. Wait until the status is `Registered` before continuing.

    ```azurepowershell-interactive
    Get-AzProviderFeature -ProviderNamespace Microsoft.NetApp -FeatureName ANFBackupByDefault
    ```
You can also use [Azure CLI commands](/cli/azure/feature) `az feature register` and `az feature show` to register the feature and display the registration status.

## Protect volumes

1. From the **Volumes** page, right-click the volume for which you want to enable backup protection and select **Data Protection**.

   
1. Specify the parameters for the volume:

    **Backup vault**      
        Specify the backup vault for the volume or [create a new backup vault](backup-vault-manage.md). 
        
    **Backup policy**  
        Specify the backup policy for the volume or [create a new backup policy](backup-configure-policy-based.md). 

    **Policy state**  
        Select the state of the backup policy.  

    **Daily backups retained**  
        Specifies the number of backups that can be retained on a daily basis.

    **Weekly backups retained**  
        Specifies the number of backups that can be retained on a weekly basis. 

    **Monthly backups retained**  
        Specifies the number of backups that can be retained on a monthly basis.

1. Click **Ok**.