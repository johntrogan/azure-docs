---
title: "Quickstart: Get started with Microsoft Discovery"
description: Set up Microsoft Discovery by creating a workspace, supercomputer, agents, and workflows, then run your first AI-powered scientific investigation.
author: mukesh-dua
ms.author: mukeshdua
ms.service: azure
ms.topic: quickstart
ms.date: 03/06/2026
ms.custom:
  - template-quickstart

#customer intent: As a scientist or engineer, I want to set up Microsoft Discovery so that I can run AI-powered scientific investigations.

---

# Quickstart: Get started with Microsoft Discovery

In this quickstart, you set up your Microsoft Discovery environment and run your first AI-powered scientific investigation. You complete these tasks:

- Set up prerequisites (resource providers, roles, networking, identity, and storage)
- Create a supercomputer
- Create a workspace
- Log in to Microsoft Discovery Studio
- Create a project
- Create required agents
- Create an investigation under your project
- And then start a chat in your investigation

## Prerequisites

- An **active [Azure subscription](https://portal.azure.com/)** that is enabled for Microsoft Discovery **public preview support**.
- **Sufficient permissions** in your Azure subscription to register resource providers and create resources:
  - The **Owner** or **Role Based Access Control Administrator** or **User Access Administrator** role is required to assign roles to administrators (Platform Admins, Scientists, and Engineers) who manage and use Discovery resources. For more information, see [Assign roles to administrators](#1b-assign-roles-to-administrators).
- **Microsoft Foundry, Azure OpenAI quotas, and VM SKU/quotas** available in your chosen region.
- An existing **resource group**, or permissions to [create a new one](https://learn.microsoft.com/azure/azure-resource-manager/management/manage-resource-groups-portal). Creating a resource group requires the Contributor role on the subscription.
- A **virtual network and subnets** for your workspace and supercomputer. See [Create a virtual network and subnets](#1c-create-a-virtual-network-and-subnets).
- **User Assigned Managed Identities (UAMI)** with the required Azure role assignments for your supercomputer, workspace, and Azure Blob Storage. See [Create a User Assigned Managed Identity (UAMI)](#1d-create-user-assigned-managed-identity-uami).

> [!IMPORTANT]
> Microsoft Discovery resources are supported in four production regions: **East US**, **East US 2**, **Sweden Central**, and **UK South**. Create all resources for a single deployment in the same region, subscription, and resource group for simplicity.

### 1a. Register resource providers

To register a resource provider in your Azure subscription, you need to have a Contributor or higher privileged role (for example, Owner) and follow these steps:

1. Sign in to the [Azure portal](https://portal.azure.com).
1. Navigate to **Subscriptions** and select your subscription.
1. In the left-hand menu, select **Resource Providers**.
1. Search for `Microsoft.Discovery`.
1. Select the provider name and select **Register**.

> [!NOTE]
> Ensure that the following resource providers are also registered on your subscription. If not, register them:
> `Microsoft.Network`, `Microsoft.Compute`, `Microsoft.Storage`, `Microsoft.ManagedIdentity`, `Microsoft.AlertsManagement`, `Microsoft.Authorization`, `Microsoft.CognitiveServices`, `Microsoft.ContainerInstance`, `Microsoft.ContainerRegistry`, `Microsoft.ContainerService`, `Microsoft.DocumentDB`, `Microsoft.Features`, `Microsoft.KeyVault`, `Microsoft.MachineLearningServices`, `Microsoft.OperationalInsights`, `Microsoft.ResourceGraph`, `Microsoft.Search`, `Microsoft.Web`, `Microsoft.Insights`, `Microsoft.Resources`, `Microsoft.Sql`, `Microsoft.App`, `Microsoft.Bing`

### 1b. Assign roles to administrators

Assign the following built-in roles to users at the desired scope (subscription or resource group):

- Microsoft Discovery Platform Administrator (Preview)
- Managed Identity Contributor
- Managed Identity Operator
- Storage Account Contributor
- Storage Blob Data Contributor
- Network Contributor
- ACRPush

**Steps to assign roles:**

1. Sign in to the [Azure portal](https://portal.azure.com).
1. Navigate to **Subscriptions** and select your subscription.
1. In the left-hand menu, select **Access control (IAM)**.
1. Select **Add**, then select **Add role assignment**.
   :::image type="content" source="./media/assign-role.jpg" alt-text="Screenshot showing the Add role assignment option in Access control (IAM)."::: # TBD
1. On the **Add role assignment** pane, search for each role listed above **one role at a time**, then select **Next**.
1. On the **Members** tab, ensure **Assign access to** is set to **User, group, or service principal**.
1. Select **+ Select members**, choose the members to assign this permission to, then select **Next**.
   :::image type="content" source="media/assign-role-members.jpg" alt-text="Screenshot showing the Members tab for adding role assignment members."::: #TBD
1. On the **Conditions** tab, select **Allow user to assign all roles except privileged administrator roles Owner, UAA, RBAC (Recommended)**, then select **Next**.
1. On the **Assignment Type** tab, select the configuration that best suits your organization, then select **Next**.
1. On the **Review + assign** tab, verify all the information and select **Review + assign**.

Repeat this process for all roles listed above.

### 1c. Create a virtual network and subnets

1. Sign in to the [Azure portal](https://portal.azure.com).
1. Search for **Virtual networks** and select it from the results.
1. Select **Create** to start creating a new virtual network.
1. Enter details such as Subscription, Resource Group, Name, and Region, then select **Next**.
1. Configure IP addresses:
   - **IPv4 address space**: Enter your chosen CIDR block (for example, `10.0.0.0/16`).
   - Add the following subnets:
     - `supercomputerNodepoolSubnet`: `10.0.1.0/24`
     - `aksSubnet`: `10.0.2.0/24`
     - `workspaceSubnet`: `10.0.3.0/24`
     - `privateEndpointSubnet`: `10.0.4.0/24`
     - `agentSubnet`: `10.0.5.0/24`    # TBD - how should this be managed?
1. Review and create the virtual network.
   :::image type="content" source="./media/create-vnet-1.jpg" alt-text="Screenshot of the Create virtual network page showing IP address configuration.":::    # TBD
1. On the **Subnets** page, select **workspaceSubnet**.
1. In the **Edit subnet** pane, scroll down to **Subnet Delegation**, search for `Microsoft.App/environments`, and select it.
1. Select **Save**.

> [!NOTE]
> Network Security Groups (NSGs) aren't required for this step, but it's a general best practice to implement NSGs for each subnet in a virtual network, depending on your organization's policies.

### 1d. Create a user assigned managed identity (UAMI)

You can create different UAMIs each with their own required permissions for specific resource access, or you can create a single UAMI with all necessary permissions for the platform. For this exercise, create a single UAMI by following these steps:

1. Sign in to the [Azure portal](https://portal.azure.com).
1. Search for **Managed Identities** and select it from the list.
1. Select **Create**.
1. Fill in the required details such as subscription, resource group, region, and name.
1. Select **Review + Create**, then select **Create**.

Assign the following built-in roles to the new User Assigned Managed Identity at Resource Group level:

- Microsoft Discovery Platform Contributor (Preview)
- Storage Blob Data Contributor
- ACRPull

1. Navigate to **Subscriptions** and select your subscription.
1. Select the resource group that you are using for this exercise.
1. In the left-hand menu, select **Access control (IAM)**.
1. Select **Add**, then select **Add role assignment**.
1. On the **Add role assignment** pane, search for each role listed above **one role at a time**, then select **Next**.
1. On the **Members** tab, ensure **Assign access to** is set to **Managed Identity**.
1. Select **+ Select members**. In the **Select managed identities** pane, select your subscription, select **User-assigned managed identity** type, select your managed identity we created in the previous step, then select **Select**.
1. On the **Review + assign** tab, verify all the information and select **Review + assign**.

:::image type="content" source="./media/assign-roles-uami-1.jpg" alt-text="Screenshot of the Azure portal showing UAMI role assignment.":::

### 1e. Create an Azure Blob Storage account

To store input and output data for your investigations, create an Azure blob storage account to associate with your storage container or use an existing one with the following requirements:

- The storage account must allow access from the Virtual Network used to create the supercomputer and workspace.
- The storage account must allow access from your client public IP or local network so you can access the output data.
- The storage account must have the correct CORS settings. You must allow these origins: `https://studio.discovery.microsoft.com`, `https://vscode.dev`, and `https://*.vscode-cdn.net`. Set the allowed operations to include `GET`, `HEAD`, `DELETE`, and `PUT` and set `Allowed Headers` and `Exposed Headers` to `*`, and `Max Age` to `200`. This setting is found under the **Resource sharing (CORS)** page under the **Settings** tab.
- Ensure that the storage account has `Storage Blob Data Contributor` access to the UAMI created in the [previous step](#1d-create-user-assigned-managed-identity-uami).

**To create an Azure blob storage account:**

1. Sign in to the [Azure portal](https://portal.azure.com).
1. Search for **Storage accounts** and select it from the results.
1. Select **Create** to start creating a new storage account.
1. Enter details such as Subscription, Resource Group, Name, and Region.
1. Select **Azure Blob Storage** as the primary service, then select the **Networking** tab.
1. Under public network access, select **Enable public access from selected virtual networks and IP addresses**.
1. Select the Virtual Network and all subnets created in [step 1c](#1c-create-a-virtual-network-and-subnets).
1. Select **Add your client IP address** if you're accessing data over the internet, or ensure your client can access the storage account and VNet via private link, Site-to-Site VPN, or ExpressRoute.
   :::image type="content" source="./media/create-storage-blob-4.jpg" alt-text="Screenshot showing the networking configuration for the storage account."::: #TBD
1. Select **Review + create**, then select **Create**.

> [!NOTE]
> To view and download output files, your client/browser needs network access to the blob storage. You can allow public internet access (either open public access to all or allow your client's public IP address in the storage networking and firewall settings), or configure private access via Azure VPN or ExpressRoute.

#### Enable CORS and UAMI access

1. Open the storage account we created in the previous step.
1. Under the **Settings** tab, select **Resource sharing (CORS)**.
1. Under **Blob service** in the **Allowed origins** column, enter `https://studio.discovery.microsoft.com`, `https://vscode.dev` and `https://*.vscode-cdn.net`. For both, set the allowed operations to include `GET`, `HEAD`, `DELETE`, `OPTIONS`, and `PUT`. Set `Allowed Headers` and `Exposed Headers` to `*`, and `Max Age` to `200`.
1. Select **Save**.
   :::image type="content" source="media/create-storage-blob-3.jpg" alt-text="Screenshot showing the CORS configuration for the storage account blob service."::: # TBD

## 2. Create a supercomputer

To deploy and run scientific tools, index your data in Bookshelf knowledge bases, and execute GPU/CPU-intensive workloads for simulation and modeling, you need a supercomputer with associated node pools. The supercomputer provides the compute resources on a specific virtual network within your subscription.

1. Sign in to the [Azure portal](https://portal.azure.com).
1. Search for **Microsoft Discovery Supercomputers**.
1. Select **Create** and enter details such as Subscription ID, Resource Group name, Location, and Name, then select **Next**.
   :::image type="content" source="./media/create-supercomputer-1.jpg" alt-text="Screenshot showing the basic details page for creating a Microsoft Discovery Supercomputer.":::  # TBD
1. On the **Networking** tab, select the Virtual Network and `aksSubnet` created in [step 1c](#1c-create-a-virtual-network-and-subnets), then select **Next**.
   :::image type="content" source="./media/create-supercomputer-1-5.jpg" alt-text="Screenshot showing the networking configuration for the supercomputer.":::    # TBD
1. On the **Encryption** tab, select Microsoft-managed keys and select **Next**
1. Add the User Assigned Managed Identity (UAMI) created in [step 1d](#1d-create-user-assigned-managed-identity-uami) for the cluster identity, kubelet identity, and workload identity. Supercomputer instances use this managed identity to access data from your Azure resources.
   :::image type="content" source="./media/create-supercomputer-2.jpg" alt-text="Screenshot showing the identity configuration step for the supercomputer.":::    # TBD
   :::image type="content" source="./media/create-supercomputer-2-5.jpg" alt-text="Screenshot showing the UAMI assigned to the supercomputer.":::    # TBD
1. Add tags as needed, and move to next tab. 
   > [!IMPORTANT]
   > Do not remove the version tag that is pre-populated.
1. Review the Terms and Conditions, then select **Create**.
   :::image type="content" source="./media/create-supercomputer-3.jpg" alt-text="Screenshot of the Microsoft Discovery Supercomputer overview page after creation.":::    # TBD

### 2a. Create node pools

After your supercomputer is created, follow these steps to create a node pool:

1. Open your Supercomputer in the Azure portal.
1. In the left pane, select **Nodepool** under **Settings**, then select **Create**.
   :::image type="content" source="./media/create-supercomputer-nodepool-1.jpg" alt-text="Screenshot showing the create nodepool option in the supercomputer settings.":::    # TBD
1. Enter the name and location for the node pool, then select **Next**.
   > [!NOTE]
   > Nodepool names must be all lowercase, a maximum of 12 characters, must start with a letter, and can only contain letters and numbers.
1. On the **Networking** tab, select the Virtual Network and `supercomputerNodepoolSubnet` created in [step 1c](#1c-create-a-virtual-network-and-subnets). This must be the same virtual network selected for the storage in [step 2](#2-create-shared-storage), then select **Next**.
1. On the **VM configuration** tab, select the Virtual Machine SKU to use for the nodepool, then select **Next**. The selected SKU and quota must be available in the region where you deploy the nodepool.
   :::image type="content" source="./media/create-supercomputer-nodepool-2.jpg" alt-text="Screenshot showing the VM SKU selection for the nodepool.":::    # TBD
1. In the **Scaling** section, select the maximum node count that your nodepool can scale to.
   :::image type="content" source="./media/create-supercomputer-nodepool-3.jpg" alt-text="Screenshot showing the scaling configuration for the nodepool.":::    #TBD
1. Add tags as needed, and move to next tab. 
   > [!IMPORTANT]
   > Do not remove the version tag that is pre-populated.
1. Review the Terms and Conditions, then select **Create**.

## 3. Create a workspace

A workspace is a collaborative environment where teams manage large-scale scientific initiatives. Workspaces bring together the infrastructure resources such as supercomputers, agents, tools, and knowledge bases (Bookshelves) into a single secure boundary. You can create projects under workspaces, allowing researchers to organize experiments, analyze data, and use AI agents within a shared space.

> [!IMPORTANT]
> Make sure your workspace name is globally unique and uses only lowercase letters.

1. Sign in to the [Azure portal](https://portal.azure.com).
1. Search for **Microsoft Discovery Workspaces**.
1. Select **Create** and enter details such as Subscription, Resource Group, Name, and Region, then select **Next**.
   :::image type="content" source="./media/create-workspace-1.jpg" alt-text="Screenshot showing the basic details page for creating a Microsoft Discovery workspace.":::    # TBD
1. On the **Networking** tab, select "Public netwrok access" as "enabled". Followed by that populate the details for Private Endpoint subnet, Agent subnet and Workspace subnet with the subnets created earlier in Step 1, then select **Next**.
   :::image type="content" source="media/create-workspace-2.jpg" alt-text="Screenshot showing the Networking tab while creating a workspace.":::    # TBD
1. On the **Encryption** tab, select between Microsoft-Assigned or Customer-Managed Keys (CMK). For this exercise we will use Microsoft-managed keys, just select **Next** to go to next tab. 
1. On the **Supercomputer** tab, select **Add Supercomputer** and select your subscription, resource group, and the supercomputer created in [step 2](#2-create-a-supercomputer), then select **Next**.
   :::image type="content" source="media/create-workspace-3.jpg" alt-text="Screenshot showing the Supercomputer tab while creating a workspace.":::    # TBD
1. On the **Workspace Identity** tab, select **Add** under **User Assigned Managed Identity (UAMI)** and select the identity created in [step 1d](#1d-create-user-assigned-managed-identity-uami) to provide access to the workspace.
   :::image type="content" source="media/create-workspace-4.jpg" alt-text="Screenshot showing the Workspace Identity tab with the UAMI added.":::    # TBD
1. Add tags as needed, and move to next tab. 
   > [!IMPORTANT]
   > Do not remove the version tag that is pre-populated.
1. Review the Terms and Conditions, then select **Create**.
   :::image type="content" source="media/create-workspace-5.jpg" alt-text="Screenshot of the Microsoft Discovery Workspace overview page after creation.":::    # TBD

After the workspace is created, you can provide access to users via [Role Based Access Control (RBAC)](https://learn.microsoft.com/azure/role-based-access-control/quickstart-assign-role-user-portal). To create projects in a workspace, you need to provide Contributor access to the user.

## 4. Create Chat Model Deployment

1. Go to the overview page of Microsoft Discovery workspace, created in the previous step. 
1. Under the **Settings** tab on left navigation pane, select **Chat Model Deployments**.
1. Select "+ Create" option on the top
1. Provide the **Model format** (only option available today is OpenAI) and **Model Name** in the drop-down. Use "gpt-4.1" for this exercise.
1. Then select "Create" button at the bottom.

## 5. Log in to Microsoft Discovery Studio

Microsoft Discovery Studio is a secure, AI-powered research environment that enables scientists and engineers to accelerate innovation through autonomous agents, simulation workflows, and integrated data tools—all within a unified interface.

After your infrastructure is set up, you can log in to [Microsoft Discovery Studio](https://studio.discovery.microsoft.com) directly via the URL, or find the URL in the Workspace overview page in the Azure portal.

:::image type="content" source="./media/studio-home.jpg" alt-text="Screenshot of the Microsoft Discovery Studio homepage."::: # TBD

You must sign in with your Entra ID (work or school account) credentials. Studio supports Single Sign-On (SSO) with Entra ID so that you don't have to explicitly provide credentials if you're already signed in to another service with your Entra ID in the same browser.

> [!NOTE]
> If you have access to multiple Entra tenants, make sure the right tenant is selected when signing in.

## 6. Create storage containers

After you sign in to the studio, create storage containers to organize and manage your storage assets used in your projects.

Storage containers store both input and output data as storage assets. Both inputs and outputs use a storage container of type Azure Storage Blob, backed by the storage account created in [step 1e](#1e-create-an-azure-blob-storage-account).

1. In **Microsoft Discovery Studio**, on the left navigation pane, select the **Data** tab.
   :::image type="content" source="media/studio-storage-containers-1.jpg" alt-text="Screenshot showing the Data Containers page in Microsoft Discovery Studio.":::    # TBD
1. **Storage Containers (new)** tab is selected by default.
1. Select **Create Container**.
1. Enter details such as name, subscription, resource group, and location.
1. Select the storage account created in [step 1e](#1e-create-an-azure-storage-storage-account).
   :::image type="content" source="media/studio-data-containers-2.jpg" alt-text="Screenshot showing the Create Storage Container page with Azure Storage Blob selected.":::    TBD
1. Select **Create**.

> [!NOTE]
> After you select **Create**, the resource is initially in the **Accepted** state. Refresh the page and wait until the **Provisioning State** changes to **Succeeded** before proceeding. This operation typically takes a few minutes.

## 7. Create a project

Projects help you organize and manage scientific investigations within a workspace. Each project defines the functional boundary for access to your agents, workflows, tools, and data containers. Within a project, you can run experiments, analyze data, apply AI models, and track research progress in a collaborative environment.

> [!IMPORTANT]
> Your project name must be all lowercase and no more than 12 characters long.    # TBD

1. In **Microsoft Discovery Studio**, on the left navigation pane, select **Projects**. This lists all existing projects across your Azure subscriptions and resource groups.
1. Select **Create Project**.
1. Enter the details such as name and select the workspace the we created in [step 3](#3-create-a-workspace).
1. For this exercise, uncheck the "Create storage container for me" option
1. Select the storage container created in [step 6](#6-create-storage-containers).
1. Select **Create**.

> [!NOTE]
> After you select **Create**, the project is initially in the **Accepted** state. Refresh the page and wait until the **Provisioning State** changes to **Succeeded** before proceeding.

## 8. Create an agent

Agents are autonomous, AI-powered systems that perform specific scientific tasks on behalf of users. Powered by large language models (LLMs), agents can use tools, models, and other agents to achieve a goal. In the Microsoft Discovery architecture, agents are the primary functional unit of execution.

In this example, create a basic Chemistry Agent that answers questions about chemical properties of molecules and provides a plan to calculate any property.

1. Sign in to [Microsoft Discovery Studio](https://studio.discovery.microsoft.com/).
1. In the Welcome tab, under **Get started** section, select **Create an agent** button.
1. In the **New Agent** tab,select **Agent** as the type.
1. Enter a **Name** and **Description** for the agent. For example:
   - **Name**: `ChemistryAgent`
   - **Description**: `A chemistry expert agent that answers questions about chemical properties of molecules and provides high-level plans for computational needs.`
1. Under **Chat model**, select the model deployment created in [step 4](#4-create-chat-model-deployment).
1. Enter the agent **Instructions**. For example:
   ```
   You are a chemistry expert agent who can answer questions about chemical properties of molecules and provide high-level plans for the user's computational needs.
   ```
1. Select **Create agent**.
:::image type="content" source="./media/studio-create-agent-1.jpg" alt-text="Screenshot showing the Create Agent page in Discovery Studio.":::

## 9. Create an investigation

Investigations are research studies within a project where you can chat with your agents, run computational analyses, and get data-driven insights to scientific questions.

> [!IMPORTANT]
> Your investigation name must be no more than 20 characters long.

1. In the left navigation pane under the **Investigations** tab, select the **Create investigation** button (**+**).
1. Provide a name and an optional description, then select **Create**.
   :::image type="content" source="./media/studio-investigations-1.jpg" alt-text="Screenshot showing the Create investigation dialog in Microsoft Discovery Studio.":::

## 10. Start a chat

After your investigation is created, follow these steps:

1. Select the investigation created in [step 9](#9-create-an-investigation) to open it.
1. In the chat input box, select the agent that we just created in [step 8](#8-create-an-agent)
1. Enter a prompt and select **Send** to get a response using the agent configured in this quickstart.
   :::image type="content" source="./media/studio-investigations-chat-1.jpg" alt-text="Screenshot showing the chat with Copilot interface in a Microsoft Discovery investigation.":::

## Related content

- [What is Microsoft Discovery?](overview-what-is-microsoft-discovery.md)