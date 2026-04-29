---
title: Migrate machines as physical servers to Azure with Azure Migrate and Modernize
description: This article describes how to migrate physical machines to Azure with Azure Migrate and Modernize.
author: dhananjayanr98
ms.author: dhananjayanr
ms.manager: kmadnani
ms.topic: tutorial
ms.service: azure-migrate
ms.reviewer: v-uhabiba
ms.date: 08/26/2025
ms.custom: MVC, engagement-fy25

# Customer intent: "As an IT administrator planning a cloud migration, I want to migrate physical servers to Azure using a replication appliance, so that I can leverage cloud resources for improved efficiency and scalability."
---


# Migrate machines as physical servers to Azure

This article shows you how to migrate machines as physical servers to Azure by using the Azure Migration and modernization tool. Migrating machines by treating them as physical servers is useful in many scenarios:

- Migrate on-premises physical servers.
- Migrate virtual machines (VMs) virtualized by platforms such as Xen and KVM.
- Migrate Hyper-V or VMware VMs, if you're unable to use the standard migration process for [Hyper-V](tutorial-migrate-hyper-v.md) or [VMware](server-migrate-overview.md) agentless migration.
- Migrate VMs running in private clouds.
- Migrate VMs running in public clouds, such as Amazon Web Services (AWS) or Google Cloud Platform (GCP).

 In this tutorial, you learn how to:

> [!div class="checklist"]
> * Prepare Azure with required permissions to use Azure Migrate.
> * Check requirements for machines you want to migrate. Prepare a machine for the Azure Migrate and Modernize replication appliance that's used to discover and migrate machines to Azure.
> * Set up the replication appliance.
> * Install the Mobility service on machines you want to migrate.
> * Start migration execution.
> * Track and monitor migrations.
> * Run a test migration to make sure everything's working as expected.
> * Run a full VM migration.

> [!NOTE]
> Tutorials show you the simplest deployment path for a scenario so that you can quickly set up a proof of concept. Tutorials use default options where possible and don't show all possible settings and paths. For detailed instructions, review the how-to articles for Azure Migrate and Modernize.

If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/pricing/free-trial/) before you begin.

## Prerequisites

Before you begin this tutorial, you should:

1. Go to the already created project or [create a new project](create-manage-projects.md).
2. [Complete the first tutorial](tutorial-discover-physical.md) to prepare Azure and discover physical servers for migration.
3. We recommend that you complete the second tutorial to [assess physical servers](how-to-create-assessment.md) before migrating them to Azure.
4. Verify permissions for your Azure account - Your Azure account needs permissions to create a VM, and write to an Azure managed disk.
5.  For the required Azure Migrate built‑in roles and permission details to create a project and run discovery, assessments, and migrations, see [Prepare Azure accounts for Azure Migrate](prepare-azure-accounts.md).
6.  Assign permissions to register the Replication Appliance in Microsoft Entra ID. [Learn more](../site-recovery/deploy-vmware-azure-replication-appliance-modernized.md#required-permissions).

Also,

- [Review](./agent-based-migration-architecture.md) the migration architecture.
- [Review](../site-recovery/migrate-tutorial-windows-server-2008.md#limitations-and-known-issues) the limitations related to migrating Windows Server 2008 servers to Azure.

[!INCLUDE [end-of-life-notes-windows-server-2008.md](./includes/end-of-life-notes-windows-server-2008.md)]

> [!NOTE]
> If you're planning to upgrade your Windows operating system, Azure Migrate and Modernize might download the Windows SetupDiag utility for error details in case upgrade fails. Ensure that the VM created in Azure after the migration has access to [SetupDiag](https://go.microsoft.com/fwlink/?linkid=870142). If there's no access to SetupDiag, you might not be able to get detailed OS upgrade failure error codes but the upgrade can still proceed.

### Create an Azure network

> [!IMPORTANT]
> Virtual networks are a regional service, so make sure you create your virtual network in the desired target Azure region. For example, if you're planning on replicating and migrating VMs from your on-premises environment to the East US Azure Region, your target virtual network *must be created* in the East US Region. To connect virtual networks in different regions, see [Virtual network peering](../virtual-network/virtual-network-peering-overview.md).

[Set up](../virtual-network/manage-virtual-network.yml#create-a-virtual-network) an Azure virtual network. When you replicate to Azure, Azure VMs are created and joined to the Azure virtual network that you specified when you set up migration.

## Prepare for migration

To prepare for physical server migrations, you need to:
    - **Check machine requirements**: Your source machines are supported for physical server migration.
	- **Set up a replication appliance**: Physical servers require a separate replication appliance to orchestrate agent-based migrations. The Azure migrate appliance used for discovery cannot be used to execute physical server migrations. 

> [!NOTE]
> Azure migrate appliance based discovery is a prerequisite to set up replication appliance and track agent-based migrations in the new portal. For standalone migration execution using replication appliance, redirect to the classic portal by using the link available in your **Azure Migrate Project>Execute>Migrations**.

### Check machine requirements for migration

Make sure machines comply with requirements for migration to Azure.

> [!NOTE]
> When you migrate physical machines, the Migration and modernization tool uses the same replication architecture as agent-based disaster recovery in Azure Site Recovery. Some components share the same code base. Some content might link to Site Recovery documentation.

1. [Verify](migrate-support-matrix-physical-migration.md#physical-server-requirements) physical server requirements.
1. Verify that on-premises machines that you replicate to Azure comply with [Azure VM requirements](migrate-support-matrix-physical-migration.md#azure-vm-requirements).
1. Some changes are needed on VMs before you migrate them to Azure:

    - For some operating systems, Azure Migrate and Modernize makes these changes automatically.
    - Make these changes before you begin migration. If you migrate the VM before you make the change, the VM might not boot up in Azure.
Review [Windows](prepare-for-migration.md#windows-machines) and [Linux](prepare-for-migration.md#linux-machines) changes you need to make.

### Prepare a machine for the replication appliance

The Azure Site Recovery Replication appliance is used to replicate machines to Azure. [Learn more](../site-recovery/physical-server-azure-architecture-modernized.md).

To set up a new appliance, you can use PowerShell installer script. Ensure you meet the [hardware](../site-recovery/replication-appliance-support-matrix.md#hardware-requirements) and [software](../site-recovery/replication-appliance-support-matrix.md#software-requirements) requirements, and any other prerequisites. 

> [!NOTE]
> The replication appliance shouldn't be installed on a source machine that you want to replicate or on the Azure Migrate: Discovery and assessment appliance you might have installed before.

## Set up the replication appliance

> [!NOTE]
> Classic replication appliance is retiring on **30 September 2026**. Final recovery point for existing replications will be on **31 May 2026**. Support for migrating these will continue until the retirement date (**30 September 2026**). You need to switch to the simplified appliance for all new migrations using the agent-based replication appliance.

This section describes how to set up the simplified appliance using PowerShell installer script.

1. In the Azure Migrate project > **Execute** > **Migration**, select **Start execution**.
2. In **Specify intent**, > **What do you want to migrate**, select **Servers or Virtual Machines(VM)**. Under **Where do you want to migrate to**, select **Azure VM**.
3. In **How will you select workloads**, You can either manually select servers using **From all inventory** or select an existing assessment using **From an assessment**.
4. In **Discovery method**, select the appliance that matches your source environment (Physical in this case) and click Next.
5. In **Workloads** page, Select **Set up the replication appliance** to start the appliance set up.   
6. Virtualization type will be prepopulated and greyed out based on the source azure migrate appliance type used for discovery.
7. In **Target region**, select the Azure region to which you want to migrate the machines.
8. Select **Confirm that the target region for migration is region-name**.
9. Select **Create resources**. This creates an Azure Site Recovery vault in the background.
    > [!NOTE]
    > You can't change the target region for this project after clicking this button, and all subsequent migrations are to this region.

**Set up the appliance using PowerShell**

Follow these steps:

1. [Download](https://aka.ms/V2ARcmApplianceCreationPowershellZip) the installers from the portal page or from the link provided here and place them on the replication appliance you created in your environment as per the [requirements](#prepare-for-migration).
1. Unzip and extract the components.
1. Execute the **DRInstaller.ps1 PowerShell** script as an administrator.

### Register appliance

After the appliance is created, the **Microsoft Azure Appliance Configuration Manager** launches automatically. It validates prerequisites such as internet connectivity, time synchronization, system configurations, and group policies.

**CheckRegistryAccessPolicy** - Prevents access to registry editing tools
    - Key: `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`.
    - The DisableRegistryTools value shouldn't be equal to 0.

**CheckCommandPromptPolicy** - Prevents access to the command prompt
    - Key: `HKLM\SOFTWARE\Policies\Microsoft\Windows\System`.
    - The DisableCMD value should be equal to 0.

**CheckTrustLogicAttachmentsPolicy** - Trust logic for file attachments.
    - Key: `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Attachments`.
    - The UseTrustedHandlers value shouldn't be equal to 3.

**CheckPowershellExecutionPolicy** - Turn on Script Execution.
    - PowerShell execution policy shouldn't be set to **AllSigned or Restricted**.
    - Ensure the group policy **Turn on Script Execution Attachment Manager** isn't set to Disabled or **Allow only signed scripts**.

Use the following steps to register the appliance:

1. If the appliance uses a proxy for internet access, configure the proxy settings by toggling on the 'Use proxy to connect to internet' option.
    - All Azure Site Recovery services use these settings to connect to the internet.
    
    > [!Note]
    > Only HTTP proxy is supported.

1. Ensure the [required URLs](../site-recovery/replication-appliance-support-matrix.md#allow-urls) are allowed and reachable from the Azure Site Recovery replication appliance to maintain continuous connectivity.
1. After the prerequisites are verified, the appliance fetch all its component information in the next step. Review the status of all components and then select **Continue**.
1. **Save** the details, and then proceed to choose the appliance connectivity method. You can select either FQDN or a NAT IP to define how communication with the appliance occurs.

    :::image type="content" source="./media/tutorial-migrate-physical-virtual-machines/select-replication-appliance-connectivity.png" alt-text=" Screenshot shows how to select replication appliance connectivity":::.

1. After saving the connectivity details, select **Continue** to proceed with registration in Microsoft Azure. 
1. Ensure the [prerequisites](../site-recovery/replication-appliance-support-matrix.md#prerequisites) are met, and then proceed with the registration.

    :::image type="content" source="./media/tutorial-migrate-physical-virtual-machines/registry-with-recovery-service-vault.png" alt-text="Screenshot shows the registry with recovery service vault.":::.
1. **Friendly name of appliance**: Provide a friendly name to track this appliance in the Azure portal under Recovery Services Vault infrastructure. 

   > [!Note]
   > The name can't be changed once set.
1. **Azure Migrate replication appliance key**: Copy the key from the portal's discovery screen.
    
    :::image type="content" source="./media/tutorial-migrate-physical-virtual-machines/generate-key.png" alt-text="Screenshot shows the generated key":::.

1. After pasting the key, select **Login**. You're redirected to a new authentication tab. By default, an authentication code is generated on the **Appliance Configuration Manager** page. Use the following code in the authentication tab. 
1. Enter your Microsoft Azure credentials to complete the registration. 
1. After successful registration, you can close the tab and return to the **Appliance Configuration Manager** to continue the setup.

    :::image type="content" source="./media/tutorial-migrate-physical-virtual-machines/microsoft-code.png" alt-text="Screenshot that shows the Microsoft code.":::

    > [!Note]
    > An authentication code expires within 5 minutes of generation. If there is inactivity for longer than this duration, you're prompted to re-log in to Azure.

1. After successfully signing in, the details for Subscription, Resource Group, and Recovery Services Vault are displayed. 
1. Select **Continue** to proceed.

    :::image type="content" source="./media/tutorial-migrate-physical-virtual-machines/register-with-recovery-services.png" alt-text="Screenshot shows how to register with recovery services":::.

1. After successful registration, proceed to configure **vCenter** details.

    :::image type="content" source="./media/tutorial-migrate-physical-virtual-machines/provide-vcenter-information.png" alt-text="Screenshot shows how to provide vcenter information":::.

1. Select **Add vCenter Server** to input the vCenter information.
1. Enter the server name or IP address of the vCenter, including the port number and then provide the username, password, and a friendly name.
This information is used to retrieve details about the virtual machines managed through the vCenter. The user account details are encrypted and stored locally on the machine. [Learn more](../site-recovery/vmware-azure-tutorial-prepare-on-premises.md).

    > [!Note]
    > If you're adding the same vCenter Server to multiple appliances, ensure that the same friendly name is used across all appliances.

1. After successfully saving the vCenter information, select **Add Virtual Machine** credentials to provide user details for the VMs discovered through the vCenter.

     > [!Note]
     > * For Linux OS, ensure to provide root credentials.
     > * For Windows OS, a user account with admin privileges should be added. These credentials are used to push the installation of the mobility agent onto the source VM during the enable replication operation. The credentials can be chosen per VM in the Azure portal during the enable replication workflow.
     > * Visit the **Appliance Configurator** to edit or add credentials to access your machines.

1. After adding the vCenter details, expand **Provide Physical Server Details** to add information about any physical servers you plan to protect.

    :::image type="content" source="./media/tutorial-migrate-physical-virtual-machines/provide-physical-server-details.png" alt-text=" Screenshot provides physical server details":::.

1. Select **Add Credentials** to add the credentials of the machine(s) you plan to protect. Provide all necessary details, such as the **Operating System**, a friendly name for the credential, **username**, and **Password**. The user account details are encrypted and stored locally on the machine.
1. Finally, select **Add**.

    :::image type="content" source="./media/tutorial-migrate-physical-virtual-machines/add-physical-server-credentials.png" alt-text="Screenshot shows how to add physical server credentials":::.

1. Select **Add Server** to add the physical server details. Provide the machine's **IP address or FQDN**
1. Select the **credential account**, and then select **Add**.

    :::image type="content" source="./media/tutorial-migrate-physical-virtual-machines/add-physical-server-details.png" alt-text="Screenshot shows how to add physical server details":::.

## Install the Mobility service agent

When you enable replication for VMware virtual machines and physical servers, the Mobility service is installed on each on-premises machine. The Mobility service captures data writes it to the machine, and forwards it to the Site Recovery process server.

You can install the Mobility service using the Mobility service agent software. The following methods are available for deployment:

- [Push installation](../site-recovery/vmware-physical-mobility-service-overview.md#push-installation): When protection is enabled via the Azure portal, Site Recovery installs the Mobility service on the server.
- Manual installation: You can install the Mobility service manually on each machine through the [user interface (UI)](../site-recovery/vmware-physical-mobility-service-overview.md#install-the-mobility-service-using-ui-modernized) or [command prompt](../site-recovery/vmware-physical-mobility-service-overview.md#install-the-mobility-service-using-command-prompt-modernized).
- [Automated deployment](../site-recovery/vmware-azure-mobility-install-configuration-mgr.md): You can automate the Mobility service installation with software deployment tools such as Configuration Manager.
  
 [Learn more](../site-recovery/vmware-physical-mobility-service-overview.md) about mobility service agent installation.

## Execute migrations

> [!NOTE]
> In the portal you can select up to 10 machines at once for replication. If you need to replicate more, then group them in batches of 10.

1. In the Azure Migrate project > **Execute** > **Migration**, select **Start execution**.

2. In **Specify intent**, > **What do you want to migrate**, select **Servers or Virtual Machines(VM)**. Under **Where do you want to migrate to**, select **Azure VM**.

3. In **How will you select workloads**, You can either manually select servers using **From all inventory** or select an existing assessment using **From an assessment**.

4. In **Discovery method**, select the appliance that matches your source environment (Physical) and click Next.

5. In **Workloads**, select the machines you want to replicate and migrate and select the **Target VM security type**. Azure Migrate supports migration to Trusted Launch Virtual Machines (TVMs). By default, it migrates eligible VMs as TVMs. These VMs provide enhanced security features such as secure boot and virtual TPM at no extra cost. We recommend using them wherever applicable.
6. Select the replication appliance you have set up from the drop-down menu or set up a new replication appliance by referring the steps provided in previous section.
7. In **Guest credentials**, In Guest credentials, select the friendly name of the credentials created previously during the replication appliance setup. Select the VMs you want to replicate and then click Next.
9. In **Target settings**, select the subscription, and target region to which you'll migrate, and specify the resource group in which the Azure VMs will reside after migration. Complete the below settings in the blade,
     - In **Availability options**, select:
          - Availability Zone to pin the migrated machine to a specific Availability Zone in the region. Use this option to distribute servers that form a multi-node application tier across Availability Zones. If you select this option,                 you'll need to specify the Availability Zone to use for each of the selected machine in the Compute tab. This option is only available if the target region selected for the migration supports Availability Zones
          - Availability Set to place the migrated machine in an Availability Set. The target Resource Group that was selected must have one or more availability sets in order to use this option.
          - No infrastructure redundancy required option if you don't need either of these availability configurations for the migrated machines.
     - In **Virtual Network**, select the Azure VNet/subnet to which the Azure VMs will be joined after migration.  
     - In  **Cache storage account**, keep the default option to use the cache storage account that is automatically created for the project. Use the dropdown if you'd like to specify a different storage account to use as the cache storage         account for replication. <br/>

   > [!NOTE]
   > - If you selected private endpoint as the connectivity method for the Azure Migrate project, grant the Recovery Services vault access to the cache storage account. [**Learn more**](migrate-servers-to-azure-using-private-link.md#grant-access-permissions-to-the-recovery-services-vault)
   > - To replicate using ExpressRoute with private peering, create a private endpoint for the cache storage account. [**Learn more**](migrate-servers-to-azure-using-private-link.md#create-a-private-endpoint-for-the-storage-account-1)
   
     - In **Disk encryption type**, select:
          - Encryption-at-rest with platform-managed key
          - Encryption-at-rest with customer-managed key
          - Double encryption with platform-managed and customer-managed keys

   > [!NOTE]
   > To replicate VMs with CMK, you'll need to [create a disk encryption set](/azure/virtual-machines/disks-enable-customer-managed-keys-portal#set-up-your-disk-encryption-set) under the target Resource Group. A disk encryption set object maps managed disks to a Key Vault that contains the CMK to use for SSE.
   
     - In **Azure Hybrid Benefit**:
          - Select **No** if you don't want to apply Azure Hybrid Benefit. Then click **Next**.
          - Select **Yes** if you have Windows Server machines that are covered with active Software Assurance or Windows Server subscriptions, and you want to apply the benefit to the machines you're migrating. Then click **Next**.
      
10. In **Compute**, review the VM name, size, OS disk type, and availability configuration (if selected in the previous step). VMs must conform with [Azure requirements](migrate-support-matrix-vmware-migration.md#azure-vm-requirements).

   - **VM size**: If you're using assessment recommendations, the VM size dropdown shows the recommended size. Otherwise Azure Migrate picks a size based on the closest match in the Azure subscription. Alternatively, pick a manual size in **Azure VM size**.
   - **OS disk**: Specify the OS (boot) disk for the VM. The OS disk is the disk that has the operating system bootloader and installer.
   - **Availability Zone**: Specify the Availability Zone to use.
   - **Availability Set**: Specify the Availability Set to use.
   - **Capacity reservation**: If you already have a capacity reservation for the VM SKU in the target subscription and location,         specify it here for this deployment. Capacity reservations ensure that the required VM SKU is available when you start              migration. You can associate a reservation now or skip this step and configure it later during the migration. The capacity reservation for the SKU can be in any resource group within the target subscription and location.[Learn more](/azure/virtual-machines/capacity-reservation-create).

11. In **Disks**, specify whether the VM disks should be replicated to Azure, and select the disk type (Premium v2, Ultra Disk, Standard SSD, Standard HDD, or Premium Managed disks) in Azure. Then select **Next**.
    - You can exclude disks from replication.
    - If you exclude disks, they won't be present on the Azure VM after migration.
    - You can exclude disks if the mobility agent is already installed on that server. [Learn more](../site-recovery/exclude-disks-replication.md#exclude-limitations).

12. In **Tags**, choose to add tags to your Virtual machines, Disks, and NICs.

13. In **Review and start execution**, review the settings, and select **Review and start execution** to start the initial              replication for the servers.

## Track and monitor

1. In Azure Migrate project, go to Execute > Migrations. Use **View by applications** or **View by workloads** to switch how items are grouped.

2. Replication occurs as follows: 
      - When the Start Replication job finishes successfully, the machines begin their initial replication to Azure.
	     - During initial replication, a VM snapshot is created. Disk data from the snapshot is replicated to replica managed disks in         Azure.
	     - After initial replication finishes, delta replication begins. Incremental changes to the source disks are periodically              replicated to the replica disks in Azure.

3. Execution progress is shown in Execution stage and Execution status:
      - **Execution stage**: Preparation, Testing, or Completion.
      - **Execution status**: In progress, In error, Action pending, or Completed.
  
4. Execution progress is tracked across three stages in the Execution stage:
      - **Preparation**: Servers that are enabled for replication remain in the Preparation stage while initial replication (data             replication) is in progress. You can perform **Stop, Start, Pause, & Resume** operations in this stage if required using            the drop-downs available in the server drill-down blade. After initial replication is complete, the servers move to the             Testing stage. 
      -  **Testing**: Servers for which initial replication is complete and delta replication is in progress will move to the                 Testing phase. You can choose perform test migrations on a test virtual network before the actual migration                         (recommended). You can skip the Testing stage and start migration directly by using the actions available in the                    **Completion** drop-down menu. 
      - **Completion**: Servers for which Test Migrations are completed or skipped will move to this stage. You can perform final             migrations (Cutover) for these servers. After migration is completed, perform **Complete migration** to clean up the                migration resources by using the drop-downs available in the server drill-down blade.

## Run a test migration

When delta replication begins, you can run a test migration for the VMs, before running a full migration to Azure. We highly recommend that you do this at least once for each machine, before you migrate it.

- Running a test migration checks that migration works as expected, without impacting the source (on-premises or AVS) machines, which remain operational, and continue replicating.
- Test migration simulates the migration by creating an Azure VM using replicated data (usually migrating to a non-production VNet in your Azure subscription).
- You can use the replicated test Azure VM to validate the migration, perform app testing, and address any issues before full migration.

Do a test migration as follows:

1. In Azure Migrate project, Under **Execute** > **Migrations** > select the server for which you wish to do test migration by clicking on the server name under **Workloads** column.

2. In the drill-down blade, under **Testing** drop-down, select **Start test migration**.

3. In **Test migration**, select the Azure VNet in which the Azure VM will be located during testing. We recommend you use a non-production VNet. 
4. Select the subnet to which you would like to associate each of the Network Interface Cards (NICs) of the migrated VM.

    :::image type="content" source="./media/tutorial-migrate-vmware/test-migration-subnet-selection.png" alt-text="Screenshot shows subnet selection during test migration.":::
5. You have an option to upgrade the Windows Server OS during test migration. To upgrade, select the **Upgrade available** option. In the pane that appears, select the target OS version that you want to upgrade to and select **Apply**. [Learn more](./how-to-upgrade-windows.md).
6. Once you click **Test migration**, the job starts. Monitor the status in the portal under **Execution status**. After the test migration finishes, ensure you clean up the test resources by navigating to the server and selecting **Clean up test migration** under the **Testing** drop-down.

    :::image type="content" source="./media/tutorial-migrate-vmware/clean-up-inline.png" alt-text="Screenshot of Clean up migration." lightbox="./media/tutorial-migrate-vmware/clean-up-expanded.png":::

    > [!NOTE]
    > You can now register your servers running SQL server with SQL VM RP to take advantage of automated patching, automated backup and simplified license management using SQL IaaS Agent Extension.
    >- Select the server under **Workloads** column in Execute> Migrations page. In Compute and Network settings, select checkbox associated with register with SQL IaaS extension.
    >- Select Azure Hybrid benefit for SQL Server if you have SQL Server instances that are covered with active Software Assurance or SQL Server subscriptions and you want to apply the benefit to the machines you're migrating.hs.

## Migrate VMs

After you've verified that the test migration works as expected, you can migrate the source machines.

1. In Azure Migrate project, Under **Execute** > **Migrations** > select the server for which you wish to do final migration by clicking on the server name under **Workloads** column.
2. In the drill-down blade, under **Completion** drop-down, select **Migrate**.
3. In **Migrate** > **Shut down virtual machines and perform a planned migration with no data loss**, select **Yes** > **OK**.
    - By default Azure Migrate shuts down the source VM, and runs an on-demand replication to synchronize any VM changes that occurred since the last replication occurred. This ensures no data loss.
    - If you don't want to shut down the VM, select **No**
4. You have an option to upgrade the Windows Server OS during migration. To upgrade, select the **Upgrade available** option. In the pane that appears, select the target OS version that you want to upgrade to and select **Apply**. [Learn more](./how-to-upgrade-windows.md).
5. If you already have a capacity reservation for the VM SKU in the target subscription and location, specify it here for this deployment. Capacity reservations ensure that the required VM SKU is available when you start migration. The capacity reservation for the SKU can be in any resource group within the target subscription and location. [Learn more](/azure/virtual-machines/capacity-reservation-create).
6. A migration job starts for the server. Track the job in Azure notifications.
7. After the job finishes, you can view and manage the server from the **Migrations** page which will be tracked under **Completion** stage.

## Complete the migration

1. After the migration is done, in the drill-down page of the server, under Completion drop-down, select Complete migration. This stops replication for the source machine, and cleans up replication state information for the VM.
1. Verify and [troubleshoot any Windows activation issues on the Azure VM](/troubleshoot/azure/virtual-machines/troubleshoot-activation-problems).
1. Perform any post-migration app tweaks, such as updating host names, database connection strings, and web server configurations.
1. Perform final application and migration acceptance testing on the migrated application now running in Azure.
1. Cut over traffic to the migrated Azure VM instance.
1. Remove the on-premises VMs from your local VM inventory.
1. Remove the on-premises VMs from local backups.
1. Update any internal documentation to show the new location and IP address of the Azure VMs.

## Post-migration best practices

- For increased resilience:
    - Keep data secure by backing up Azure VMs by using the Azure Backup service. [Learn more](../backup/quick-backup-vm-portal.md).
    - Keep workloads running and continuously available by replicating Azure VMs to a secondary region with Site Recovery. [Learn more](../site-recovery/azure-to-azure-tutorial-enable-replication.md).
- For increased security:
    - Lock down and limit inbound traffic access with [Microsoft Defender for Cloud - Just-in-time administration](../security-center/security-center-just-in-time.md).
    - Manage and govern updates on Windows and Linux machines with [Azure Update Manager](../update-manager/overview.md).
    - Restrict network traffic to management endpoints with [network security groups](../virtual-network/network-security-groups-overview.md).
    - Deploy [Azure Disk Encryption](/azure/virtual-machines/disk-encryption-overview) to help secure disks and keep data safe from theft and unauthorized access.
    - Read more about [securing IaaS resources](https://azure.microsoft.com/services/virtual-machines/secure-well-managed-iaas/)and [Microsoft Defender for Cloud](https://azure.microsoft.com/services/security-center/).
- For monitoring and management:
    - Consider deploying [Microsoft Cost Management](../cost-management-billing/cost-management-billing-overview.md) to monitor resource usage and spending.

## Next steps

Investigate the [cloud migration journey](/azure/architecture/cloud-adoption/getting-started/migrate) in the Cloud Adoption Framework for Azure. 
