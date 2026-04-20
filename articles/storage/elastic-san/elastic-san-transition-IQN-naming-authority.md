---
title: Transition IQN Naming Authority on Connected Volumes
description: Each Elastic SAN volume connects using an [ISCSI Qualified Name (IQN)](https://docs.oracle.com/en/storage/zfs-storage/zfs-appliance/os8-8-x/admin-guide/san-iscsi-configuration.html). Learn how to transition from the `net.windows.core` IQN naming authority to the `net.azure.storage` IQN naming authority on connected Elastic SAN volumes.
ms.service: azure-elastic-san-storage
author: ssragovicz
ms.author: ssragovicz
ms.reviewer: rogarana
ms.date: 04/15/2026
ms.topic: how-to
---

# Transition IQN naming authority

Elastic SAN volume groups issue a unique identifer called an iSCSI Qualified name (IQN) that your volumes use to connect. As of June 30, 2025, all new volume groups issue an IQN using the naming authority `net.azure.storage`. If you have volume groups that were created prior to June 30, 2025, you may be connected to volumes using the authority `net.windows.core` and we recommend you transition to the naming authority `net.azure.storage` using the steps outlined in this article.

## Prerequisites

Before you transition IQN naming authorities, please ensure the following prerequisites are met:
-  An Elastic SAN volume connected using the `net.windows.core` naming authority. 
    - You can verify your active connections by running the following commands depending on your operating system 
        - Linux:  `sudo iscsiadm -m session `
        - Windows:  `iscsicli SessionList   `  
- The `net.windows.storage` disconnect script
    - [Linux](https://github.com/Azure-Samples/azure-elastic-san/blob/main/iqnMigrationCustomDisconnectScripts/disconnect_iqnv1_linux.py)
    - [Windows PowerShell](https://github.com/Azure-Samples/azure-elastic-san/blob/main/iqnMigrationCustomDisconnectScripts/disconnect_iqnv1_windows.ps1)
- The `net.azure.storage` connect script
    - Your Azure Portal contains the connect script for the same volume using the `net.azure.storage` IQN

## Transition naming authority 

1. Stop all applications and services that are accessing the affected volumes.  

2. Verify that your volume has active connections using the `iscsiadm` or `iscsicli` command

4. Copy the appropriate disconnect script into a file on each VM connected to your affected volumes and run the script with the `net.windows.storage` IQNs.  
    - [Linux](https://github.com/Azure-Samples/azure-elastic-san/blob/main/iqnMigrationCustomDisconnectScripts/disconnect_iqnv1_linux.py): 
        - `Example Command: python3 disconnect.py [IQN A], [IQN B] `
    - [Windows PowerShell](https://github.com/Azure-Samples/azure-elastic-san/blob/main/iqnMigrationCustomDisconnectScripts/disconnect_iqnv1_windows.ps1): 
        - `Example Command: .\disconnect.ps1 [IQN A], [IQN B], [IQN C]  `

5. Restart the VM.  

6. Verify that there are no active connections.  

7. Retrieve your connect script from portal for the disconnected volume(s) and run it on the VM(s) to reattach the volume(s).  

8. Restart the VM.  

9. Verify that your volume has active connections.  

10. Start your applications and services again, resuming your workloads.  