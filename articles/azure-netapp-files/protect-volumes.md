---
title: Protect volumes
description: Learn about how you can enable backup to protect your existing volumes. 
services: azure-netapp-files
author: netapp-manishc
ms.service: azure-netapp-files
ms.topic: concept-article
ms.date: 10/22/2025
ms.author: anfdocs
---
# Protect volumes of Azure NetApp Files

You can protect your existing volume by enabling backup protection for the volume. This enhances data protection with an additional layer of protection without the need for manual setup.

## Protect volumes

1. From the **Volumes** page, right-click the volume for which you want to enable backup protection and select **Data Protection**.

    >[!NOTE]
    > Enabling scheduled backup may incur additional charges as per backup pricing. You must enable this option to proceed with enabling scheduled backup.

1. Specify the parameters for the volume:

    **Backup vault**      
        Specify the backup vault for the volume or [create a new backup vault](backup-vault-manage.md). 
        
    **Backup policy**  
        Specify the backup policy for the volume or [create a new backup policy](backup-configure-policy-based.md). You can select only an enabled backup policy. The daily, weekly, and monthly backup retention periods are displayed based on the selected backup policy.  

    **Daily backups retained**  
        Specifies the number of backups that can be retained on a daily basis.

    **Weekly backups retained**  
        Specifies the number of backups that can be retained on a weekly basis. 

    **Monthly backups retained**  
        Specifies the number of backups that can be retained on a monthly basis.

1. Click **Ok**.