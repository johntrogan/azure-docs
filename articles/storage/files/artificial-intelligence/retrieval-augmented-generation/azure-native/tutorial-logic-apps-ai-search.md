---
title: Build a RAG pipeline using Azure Files with Logic Apps and AI Search
description: Learn how to build a no-code retrieval-augmented generation (RAG) pipeline that indexes documents from Azure Files using Azure Logic Apps, Azure OpenAI, and Azure AI Search.
author: ftrichardson1
ms.service: azure-file-storage
ms.topic: tutorial
ms.date: 04/15/2026
ms.author: t-flynnr
---

# Tutorial: Build a RAG pipeline using Azure Files with Logic Apps and AI Search

**Applies to:** ✔️ SMB file shares

In this tutorial, you build a no-code retrieval-augmented generation (RAG) pipeline that automatically indexes documents stored in [Azure Files](/azure/storage/files/storage-files-introduction). You use the **[Import data wizard](/azure/search/search-import-data-portal)** in [Azure AI Search](/azure/search/search-what-is-azure-search) to create a [Logic Apps](/azure/logic-apps/logic-apps-overview) workflow that connects to your file share, generates vector embeddings with [Azure OpenAI](/azure/ai-services/openai/overview), and stores the indexed content in a search index. After indexing, you use [Azure AI Foundry](/azure/ai-studio/what-is-ai-studio) to ask natural-language questions about your documents.

> [!NOTE]
> The Import data wizard creates a **[Consumption-plan](/azure/logic-apps/logic-apps-overview#resource-environment-differences)** Logic App, which uses shared multi-tenant infrastructure.

## Architecture overview

The pipeline works as follows:

1. The Import data wizard in Azure AI Search creates a Logic Apps workflow (Consumption plan) that indexes files from your Azure file share.
1. The workflow downloads each file, splits it into chunks, and sends each chunk to Azure OpenAI for embedding.
1. The embedded chunks are stored in an Azure AI Search index.
1. You query the index through Azure AI Foundry's chat playground, which retrieves relevant chunks and generates grounded answers using a chat completion model.

## Prerequisites

- An Azure subscription. If you don't have one, [create a free account](https://azure.microsoft.com/free/).
- An SMB Azure file share containing the documents you want to index and query. [NFS is not supported](https://learn.microsoft.com/en-us/azure/search/search-file-storage-integration#prerequisites).
- The [Azure File Storage connector](https://learn.microsoft.com/en-us/connectors/azurefile/#general-limits) has a general 50 MB file size limit. However, actions that support [chunking](https://learn.microsoft.com/en-us/connectors/azurefile/#actions-that-support-chunking-feature) (such as Get file content) can handle files up to 300 MB. Chunking is enabled by default. The effective limit for the wizard's workflow depends on which limit applies — verify with your own file sizes during your walkthrough.
- A resource group for the new resources. You can use the same resource group as your storage account or [create a new one](https://portal.azure.com/#create/Microsoft.ResourceGroup). The resource group location doesn't restrict which regions you can deploy resources to — it only determines where [resource group metadata is stored](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/overview#which-location-should-i-use-for-my-resource-group). However, Microsoft recommends co-locating the resource group and its resources in the same region.
- [Owner or Contributor](https://learn.microsoft.com/en-us/azure/search/search-how-to-index-logic-apps#prerequisites) in your Azure subscription, with permissions to create resources. You also need [Microsoft.Authorization/roleAssignments/write permissions](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-portal#prerequisites) (such as Owner, User Access Administrator, or Role Based Access Control Administrator) to assign roles to the Logic App's [system-assigned managed identity](https://learn.microsoft.com/en-us/azure/logic-apps/authenticate-with-managed-identity).

## Step 1: Create an Azure OpenAI resource and deploy models

Azure OpenAI provides the embedding model that converts text into vectors, and the chat completion model that generates answers.

### Create the resource

1. Search for **Azure OpenAI** in the portal.
1. Select **+ Create**, then select **Azure OpenAI** from the enabled dropdown.
1. Select your subscription.

   > [!IMPORTANT]
   > When using [managed identity authentication](https://learn.microsoft.com/en-us/azure/logic-apps/authenticate-with-managed-identity#prerequisites), the Logic App, Azure AI Search, and Azure OpenAI resources must all reside in the **same Azure subscription**. This is required to configure managed identity role assignments.

1. Select your resource group. Use a dedicated resource group for this tutorial so you can delete all resources together when finished.
1. Select a region where at least one of the [supported embedding models](https://learn.microsoft.com/en-us/azure/search/search-how-to-index-logic-apps#supported-models) is available:

   - `text-embedding-3-small`
   - `text-embedding-3-large`
   - `text-embedding-ada-002`

   Choose a region close to you for lower latency. For regional model availability, see the [Azure OpenAI model availability table](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/models).

   > [!NOTE]
   > Verify that your chosen embedding model appears in the Foundry model dropdown for the region you selected. If it doesn't appear, the region doesn't support it — choose a different region.

1. Enter a name for your Azure OpenAI resource.
1. For **Pricing tier**, select **Standard S0**. This is the only tier available for Azure OpenAI.
1. On the **Network** tab, configure network access. The [wizard creates a Consumption-plan Logic App that doesn't support private endpoints](https://learn.microsoft.com/en-us/azure/search/search-how-to-index-logic-apps#limitations), so the **Disabled** option won't work.

   # [All networks](#tab/all-networks)

   Select **All networks**. No additional configuration is needed.

   # [Selected networks](#tab/selected-networks)

   Select **Selected networks**. In the **Firewall** section, add **all** of the Logic App's [outbound IP addresses](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-limits-and-config#outbound-ip-addresses) for your AI Search resource's region to the **Address range** field. This allows the Logic App to reach Azure OpenAI through the firewall.

   > [!IMPORTANT]
   > You must add [all the outbound IP addresses](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-limits-and-config#multitenant---outbound-ip-addresses) for the region, not just some. The Logic App can use any of its region's outbound IPs — requests from IPs not on the allowlist are blocked.

   > [!IMPORTANT]
   > The wizard creates the Logic App in the same region as your AI Search resource, which must be in one of the [12 supported regions](#region-compatibility). The outbound IPs you add here must match that region — look them up in the [Multitenant - Outbound IP addresses](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-limits-and-config#multitenant---outbound-ip-addresses) table. If your storage account has [firewall rules](https://learn.microsoft.com/en-us/connectors/azurefile/#known-issues-and-limitations), your AI Search region must also be **different** from your storage account's region.

   > [!NOTE]
   > Since the Logic App doesn't exist yet at this step, you can also start with **All networks** and switch to **Selected networks** after the wizard creates the Logic App. For details on configuring virtual networks, subnets, and firewall rules, see [Configure network security for Azure OpenAI](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/create-resource#configure-network-security).

   ---
1. Select **Next**. On the **Tags** tab, optionally add tags to categorize your resource. To learn more, see [Use tags to organize your Azure resources](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/tag-resources). Select **Next** to continue.
1. On the **Review + submit** tab, Azure runs a final validation of your configuration. After validation completes, select **Create**.

### Assign the Cognitive Services OpenAI User role

Assign yourself the [**Cognitive Services OpenAI User**](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/role-based-access-control) role on the Azure OpenAI resource. This grants permission to call the chat completion and embedding APIs from Azure AI Foundry.

1. Once the deployment has completed, select **Go to resource** from the deployment overview page.
1. In the left menu, select **Access control (IAM)**.
1. Select **+ Add** > **Add role assignment**.
1. Search for **Cognitive Services OpenAI User** and select it. Select **Next**.
1. For **Assign access to**, select **User, group, or service principal**.
1. Select **+ Select members**, search for your name, and select your account.
1. Select **Review + assign**.

   > [!NOTE]
   > If you later receive a `PermissionDenied` error in the Foundry chat playground, the error message includes the principal ID that needs this role. Assign the **Cognitive Services OpenAI User** role to that principal.

### Deploy the embedding model

1. From your Azure OpenAI resource, select **Go to Foundry portal**.
1. In the left menu, under **Shared resources**, select **Deployments**.
1. Select **+ Deploy model**, then select **Deploy base model** from the dropdown.
1. Search for and select one of the three [supported embedding models](https://learn.microsoft.com/en-us/azure/search/search-how-to-index-logic-apps#supported-models): `text-embedding-3-small`, `text-embedding-3-large`, or `text-embedding-ada-002`. This tutorial uses `text-embedding-3-small`.
1. Select **Confirm**. The deployment dialog shows a **Deployment name** — you can customize this, but using the auto-filled model name (e.g., `text-embedding-3-small`) is recommended for clarity. The remaining defaults are fine for this tutorial. To learn more about each field, see [Deploy a model](https://learn.microsoft.com/en-us/azure/foundry-classic/openai/how-to/create-resource?pivots=web-portal#deploy-a-model). Select **Deploy**.

   > [!NOTE]
   > You can't upgrade between embedding models. If you switch models later, you must re-index all your documents.

   > [!IMPORTANT]
   > Record the deployment name exactly. You need it when configuring the Import data wizard.

### Note your endpoint

1. In the Azure portal, go to your Azure OpenAI resource.
1. Select **Go to Foundry portal**.
1. In the left menu, under **Resource configuration**, copy the **Azure OpenAI endpoint** and store it in a safe place. You need this when configuring the wizard.

   > [!NOTE]
   > You don't need to save an API key for Azure OpenAI. This tutorial uses the Logic App's system-assigned managed identity for the connection to Azure OpenAI, which is more secure than key-based authentication. The storage account access key is still required for the Azure Files connection because the [Azure File Storage connector](/connectors/azurefile/) doesn't support managed identity.

## Step 2: Create an Azure AI Search resource

Azure AI Search stores the vector index and provides similarity search over your embedded document chunks.

1. From the Azure portal landing page, select **+ Create a resource**.
1. Search for **Azure AI Search**.
1. Select **Azure AI Search** from the results.
1. Select **Create**.
1. Select your subscription and resource group.
1. Enter a unique name. The name must be globally unique.
1. Select a [supported region](#region-compatibility) for the Azure Files Logic Apps connector.

   > [!IMPORTANT]
   > If your storage account has **firewall rules** or is restricted to a virtual network, the Logic App can't access the file share if both resources are in the same region. As a workaround, choose a region here that is **different** from your storage account's region. This overrides the general recommendation to co-locate resources. For more information, see [Access storage accounts behind firewalls](/azure/connectors/connectors-create-api-azurefilestorage#access-storage-accounts-behind-firewalls).

1. For **Pricing tier**, select the tier that fits your workload. Choose based on your expected index size (which depends on the number and size of documents in your file share), your authentication method, and your budget.

   > [!NOTE]
   > **Basic** is the minimum tier required for [managed identity authentication](https://learn.microsoft.com/en-us/azure/search/search-how-to-index-logic-apps#prerequisites). If your file share contains a large volume of content, choose a higher tier to accommodate the resulting index size. For full tier details, see [Service limits in Azure AI Search](https://learn.microsoft.com/en-us/azure/search/search-limits-quotas-capacity). For pricing, see [Azure AI Search pricing](https://azure.microsoft.com/pricing/details/search/).
1. The **Scale** tab configures replicas and partitions for performance and availability. The defaults are fine for this tutorial. To learn more, see [Estimate and manage capacity](https://learn.microsoft.com/en-us/azure/search/search-capacity-planning).
1. On the **Networking** tab, leave the default **public endpoint**. The [wizard creates a Consumption-plan Logic App that doesn't support private endpoints](https://learn.microsoft.com/en-us/azure/search/search-how-to-index-logic-apps#limitations), so a public endpoint is required.
1. On the **Tags** tab, optionally add tags to categorize your resource. To learn more, see [Use tags to organize your Azure resources](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/tag-resources).
1. Select **Review + create**, then **Create**.

### Enable the system-assigned managed identity on AI Search

Azure AI Foundry requires the AI Search resource to have a system-assigned managed identity enabled for data connections.

1. Once the deployment has completed, go to your Azure AI Search resource.
1. In the left menu, select **Settings** > **Identity**.
1. Under **System assigned**, switch the status to **On**.
1. Select **Save**.

### Enable RBAC authentication on AI Search

Azure AI Foundry requires role-based access control (RBAC) to connect to your AI Search index.

1. In your AI Search resource, select **Settings** > **Keys**.
1. Under **API Access Control**, select **Both** to enable both API key and RBAC authentication.
1. Select **Save**.

### Grant Azure OpenAI access to AI Search

Azure OpenAI's managed identity needs permission to read your AI Search index when Foundry queries your data.

1. In your AI Search resource, select **Access control (IAM)**.
1. Select **+ Add** > **Add role assignment**.
1. Search for **Search Index Data Reader** and select it. Select **Next**.
1. For **Assign access to**, select **Managed identity**.
1. Select **+ Select members**, find **Azure OpenAI** under Managed identity, and select your Azure OpenAI resource.
1. Select **Review + assign**.
1. Repeat the same steps to assign **Search Index Data Reader** to **yourself** — select **User, group, or service principal** instead of Managed identity, search for your name, and assign.

### Note your storage account access key and endpoint

The Import data wizard requires your storage account access key and file endpoint to connect to your Azure file share. Note these values before proceeding:

1. In the Azure portal, go to your storage account.
1. In the left menu, select **Security + networking** > **Access keys**.
1. Copy **key1** and store it in a safe place.
1. In the left menu, select **Settings** > **Endpoints**.
1. Under **File service**, copy the file endpoint URL (for example, `https://mystorageaccount.file.core.windows.net/`).

   > [!IMPORTANT]
   > Store your access key securely. Don't share it or commit it to source control.

## Step 3: Import data using the wizard

The Import data wizard in Azure AI Search creates a Logic Apps workflow that connects to your file share, chunks documents, generates embeddings, and indexes the content—all without writing code. The wizard automatically creates the search index and a Consumption-plan Logic App.

1. Go to your Azure AI Search resource in the portal.
1. On the **Overview** page, select **Import data**.

> [!NOTE]
> Verify that **Azure File Storage (using Azure Logic Apps)** appears as a connector option. If it doesn't, your AI Search resource is not in a supported region. Redeploy AI Search in a supported region from the [region compatibility table](#region-compatibility).

1. Under **Choose a data source**, select **Azure File Storage (using Azure Logic Apps)** as the connector.
1. When prompted for the storage account connection, provide your storage account name or file endpoint and access key:

   | Setting | Where to find it |
   | :--- | :--- |
   | **Storage account name or file endpoint** | The full endpoint URL including `https://`, for example `https://mystorageaccount.file.core.windows.net/`. Find this in the Azure portal under your storage account > **Settings** > **Endpoints** > **File service**. Don't use a relative URL without `https://` — it will be rejected. Alternatively, the wizard may accept just the storage account name — use whichever format the UI requests. |
   | **Access key** | In the Azure portal, go to your storage account > **Security + networking** > **Access keys** > copy **key1**. |

1. Enter a name prefix (for example, `rag-demo`). This prefix is used for both the search index and the Logic App workflow.
1. For **Indexing frequency**, select **Once** for a one-time import, or **On a schedule** if you want the workflow to re-index periodically.
1. For the authentication type between the Logic App and AI Search, select the option to use the **Logic App's system-assigned managed identity**. The wizard automatically creates the necessary role assignment.
1. Select **Next** to continue.

### Vectorize your text

1. Select your Azure OpenAI subscription and resource.
1. Select your embedding model deployment (for example, `text-embedding-3-small`).
1. For the authentication type, select the option to use the **system-assigned managed identity** for the Azure OpenAI connection.
1. Select **Next** to continue.

### Review and create

1. If the wizard offers an option to enable **[semantic ranking](/azure/search/semantic-search-overview)**, enable it. This adds a semantic configuration to your index, which Azure AI Foundry uses for improved search relevance.
1. Review the configuration summary.
1. Select **Create** to begin processing.

> [!NOTE]
> After the wizard completes, verify that a Consumption-plan Logic App was created in your resource group and that the index exists under **AI Search** > **Indexes**. If either is missing, the wizard may have failed silently — check the Logic App's run history for errors.

The wizard creates the following resources:

| Component | Location | Description |
| :--- | :--- | :--- |
| Search index | Azure AI Search | Contains indexed content with a default schema (document ID, content, and vectorized content). |
| Logic App workflow | Azure Logic Apps | A Consumption-plan workflow that ingests and indexes files from your file share. |

> [!NOTE]
> The wizard automatically created a Consumption-plan Logic App in your resource group. You can view and manage it by searching for **Logic Apps** in the Azure portal.

### Verify the index

1. After the workflow completes, go to your Azure AI Search resource.
1. In the left menu, under **Search management**, select **Indexes**.
1. Select the index created by the wizard (it uses the name prefix you specified).
1. Verify that the document count is greater than zero.
1. Select **Search explorer** and run an empty search (`*`) to see the indexed chunks.

## Step 4: Verify or add a semantic configuration

If you enabled semantic ranking in the wizard, your index should already have a semantic configuration. To verify:

1. Go to your Azure AI Search resource.
1. In the left menu, under **Search management**, select **Indexes**.
1. Select your index.
1. Select the **Semantic configurations** tab.

If a semantic configuration already exists, skip to Step 5.

If the wizard didn't add one, Azure AI Foundry requires a semantic configuration for the default search experience. Add it manually:

> [!NOTE]
> Check the actual field names in the index's **Fields** tab before proceeding. The table below assumes `documentName` and `content`, but the wizard may use different names.

1. Select **+ Add semantic configuration**.
1. Configure the following settings:

   | Setting | Value |
   | :--- | :--- |
   | **Name** | `default` |
   | **Title field** | `documentName` (or the equivalent title field in your index) |
   | **Content fields** | `content` (or the equivalent content field in your index) |
   | **Keyword fields** | Leave empty |

1. Select **Save**.

> [!NOTE]
> Adding a semantic configuration doesn't affect existing indexed data. It only changes how results are ranked at query time.

## Step 5: Chat with your documents using Azure AI Foundry

To chat with your indexed documents, you need a chat completion model deployed in Azure OpenAI. If you haven't deployed one yet, go to Azure AI Foundry > **Shared resources** > **Deployments** > **+ Deploy model** > **Deploy base model** and select a chat completion model. To compare available models, see [Azure OpenAI models](/azure/ai-services/openai/concepts/models).

1. Go to [Azure AI Foundry](https://ai.azure.com).
1. Create or open a project using your subscription and resource group.
1. In the left menu, select **Playgrounds** > **Chat**.
1. Select **Add your data**.
1. Select **Azure AI Search** as the data source.
1. For **Azure resource authentication type**, select **API key**.

> [!NOTE]
> The Azure AI Foundry "Add your data" feature connects directly from Foundry to your AI Search index. This connection is separate from the Logic App's managed identity connections configured in Step 3. The exact label for the authentication type selector may differ in the UI (for example, **Data connection** or **Azure resource authentication type**).

1. Enter your AI Search endpoint, admin key, and select your index.
1. Complete the setup wizard. The default search type uses the semantic configuration (`default`) you defined in the previous step.
1. Type a question about your documents. The system retrieves relevant chunks from the index and uses your chat completion model to generate a grounded answer with citations.

## Modify the index or workflow

After the wizard creates the index and workflow, you can customize them without breaking the indexing pipeline.

### Safe index modifications

- Add a [semantic configuration](/azure/search/semantic-how-to-configure) (as shown in Step 4)
- Add [scoring profiles](/azure/search/index-add-scoring-profiles)
- Add [spell check](/azure/search/speller-how-to-add)
- Add [synonym maps](/azure/search/search-synonyms)
- Add [suggesters](/azure/search/index-add-suggesters)

> [!IMPORTANT]
> Don't modify or delete existing fields in the wizard-generated index. The Logic App workflow depends on the original schema.

### Safe workflow modifications

You can open the Logic App in the designer to customize the workflow. To find it, search for **Logic Apps** in the Azure portal and select the Logic App the wizard created (it uses the name prefix you specified). Then select **Logic app designer** from the left menu.

> [!NOTE]
> The Logic App workflow can only be managed from Azure Logic Apps, not from the AI Search portal. There's no programmatic support for managing the workflow through Azure AI Search APIs.

You can make the following changes:

- Modify the **List files in folder** step to change which documents are sent to indexing.
- Modify the **Chunk Text** step to vary the token size (512 tokens is recommended for most scenarios) or add a page overlap length. Don't exceed 8,192 tokens per chunk — that's the maximum input for the embedding models.
- Modify the **Index multiple documents** step to control the indexing schedule (if you chose scheduled indexing).

For the full list of safe modifications, see [Modify existing objects](/azure/search/search-how-to-index-logic-apps#modify-existing-objects).

For a full list of available connector actions (such as Copy file, Delete file, and Get file metadata), see [Azure File Storage connector actions](/connectors/azurefile/#actions).

## Cost estimate

This tutorial creates billable Azure resources. Costs vary by region, usage, and pricing tier. For pricing details on each service, see [Azure pricing](https://azure.microsoft.com/pricing/). To estimate costs for your specific configuration, use the [Azure pricing calculator](https://azure.microsoft.com/pricing/calculator/).

For a short-lived demo, delete the resource group immediately after testing to avoid ongoing charges.

## Clean up resources

To avoid ongoing charges, delete the resource group:

```azurecli
az group delete --name rg-logic-apps-rag-demo --yes --no-wait
```

> [!NOTE]
> Your Azure file share may be shared infrastructure—confirm with your administrator before deleting.

## Region compatibility

This tutorial uses four Azure services. Each service has its own region requirements, and the resources don't all need to be in the same region. Choose regions based on which features you need and your proximity for lower latency. For official model availability, see [Azure OpenAI models](/azure/ai-services/openai/concepts/models). For wizard region support, see [Logic Apps connector for Azure AI Search](/azure/search/search-how-to-index-logic-apps).

> [!NOTE]
> Select regions close to your location for lower latency on portal and API interactions. For example, if you're on the US West Coast, West US 3 is a good choice. See [resource group location guidance](/azure/azure-resource-manager/management/overview#which-location-should-i-use-for-my-resource-group) for more details.

The following table shows which Import data wizard regions also support Azure OpenAI embedding models (`text-embedding-3-small`, `text-embedding-3-large`, `text-embedding-ada-002`) and `gpt-4o` via Global Standard deployment:

| Region | Import data wizard | Azure OpenAI embeddings | gpt-4o / gpt-4o-mini | All services co-located |
| :--- | :---: | :---: | :---: | :---: |
| **East US** | ✅ | ✅ | ✅ | ✅ |
| **East US 2** | ✅ | ✅ | ✅ | ✅ |
| **South Central US** | ✅ | ✅ | ✅ | ✅ |
| **West US 3** | ✅ | ✅ | ✅ | ✅ |
| **Brazil South** | ✅ | ✅ | ✅ | ✅ |
| **Australia East** | ✅ | ✅ | ✅ | ✅ |
| **Sweden Central** | ✅ | ✅ | ✅ | ✅ |
| **UK South** | ✅ | ✅ | ✅ | ✅ |
| **West US 2** | ✅ | ⚠️ | ⚠️ | ❌ |
| **East Asia** | ✅ | ⚠️ | ⚠️ | ❌ |
| **Southeast Asia** | ✅ | ❌ | ❌ | ❌ |
| **North Europe** | ✅ | ⚠️ | ⚠️ | ❌ |

> [!NOTE]
> The ⚠️ entries (West US 2, East Asia, North Europe) were not listed in the Global Standard model availability table as of April 2026. The ✅ entries were verified at the same time but model availability can change. Check the [Azure OpenAI models page](/azure/ai-services/openai/concepts/models) and the portal's model deployment dropdown to confirm current availability before selecting any region.

**Legend:**
- ✅ = Available via Global Standard deployment
- ⚠️ = Not listed in the Global Standard model availability table. Azure OpenAI may need to be deployed in a different region. Check [Azure OpenAI models](/azure/ai-services/openai/concepts/models) for current availability.
- ❌ = Not available via Global Standard deployment

> [!IMPORTANT]
> Your Azure OpenAI resource and AI Search resource don't need to be in the same region. If you choose a wizard-supported region that lacks Azure OpenAI models (like West US 2), deploy Azure OpenAI in a nearby region that supports the models (like West US 3). Azure Files (your storage account) can be in any region.

For the simplest setup with all resources in one region, choose from the first eight rows: **East US**, **East US 2**, **South Central US**, **West US 3**, **Brazil South**, **Australia East**, **Sweden Central**, or **UK South**.

## Known limitations

- The wizard generates a fixed index schema (document ID, content, and vectorized content) with text extraction only. You can add elements like semantic configurations or scoring profiles, but you can't change existing fields.
- Vectorization supports text embedding only.
- Deletion detection isn't supported. You must manually delete orphaned documents from the index.
- Duplicate documents in the search index are a known issue in this preview. Consider deleting objects and starting over if this becomes an issue.
- The wizard creates a Consumption-plan Logic App. Private endpoints aren't supported for Consumption-plan workflows created by the wizard.
- The workflow uses a fixed chunking strategy. You can customize chunk size and overlap in the Logic App designer after creation.
- The Azure File Storage connector has a general [50 MB file size limit](/connectors/azurefile/#general-limits) per file, but actions that support [chunking](/connectors/azurefile/#actions-that-support-chunking-feature) can handle files up to 300 MB.
- The connector has a [throttling limit](/connectors/azurefile/#throttling-limits) of 600 API calls per connection per 60 seconds and a bandwidth limit of 1,000 MB per 60 seconds. For file shares with hundreds of files, indexing may take multiple runs to complete.
- If your storage account has firewall rules enabled, the Logic App can't access the storage account if both resources are in the same region. Deploy them in different regions as a workaround.
- Each Logic App is deployed into a single Azure region. If that region becomes unavailable, the Logic App is also unavailable. You can't move a Logic App to another region — you must re-run the Import data wizard to create a new workflow in a different region. For production workloads that require higher resiliency, see [Reliability in Azure Logic Apps](/azure/logic-apps/reliability-logic-apps).

## Related content

- [What is retrieval-augmented generation?](../overview.md)
- [Use an Azure Logic Apps workflow for automated indexing in Azure AI Search](/azure/search/search-how-to-index-logic-apps)
- [Azure Logic Apps documentation](/azure/logic-apps/logic-apps-overview)
- [Azure AI Search documentation](/azure/search/search-what-is-azure-search)
- [Azure OpenAI documentation](/azure/ai-services/openai/overview)
- [Azure AI Foundry documentation](/azure/ai-studio/what-is-ai-studio)
- [Index data from Azure Files](/azure/search/search-file-storage-integration) (REST API alternative to the Import data wizard)

## Questions

**Question for AI Search team: What file types does the Import data wizard support when using the Azure Files Logic Apps connector?**

**Background / How I got here:**

The [Index data from Azure Files](https://learn.microsoft.com/en-us/azure/search/search-file-storage-integration) page (built-in REST API indexer) lists these supported document formats:

> CSV, EML, EPUB, GZ, HTML, JSON, KML, Markdown, Microsoft Office formats (DOCX/DOC/DOCM, XLSX/XLS/XLSM, PPTX/PPT/PPTM, MSG), Open Document formats (ODT, ODS, ODP), PDF, Plain text, RTF, XML, ZIP

However, that list applies to the **built-in Azure Files indexer** (REST API approach using `"type": "azurefile"`), which has its own document cracking pipeline.

The **Import data wizard** with the Azure Files Logic Apps connector uses a different pipeline:
- The [Azure File Storage connector](https://learn.microsoft.com/en-us/connectors/azurefile/) retrieves raw file content as binary
- The Logic Apps workflow uses **Text Split** (a skillset action) to chunk the content
- The wizard docs say: *"The search index is generated using a fixed schema (document ID, content, and vectorized content), with text extraction only."* — [Import data wizard](https://learn.microsoft.com/en-us/azure/search/search-import-data-portal)

The wizard also uses skillsets:
- Source: *"Document Layout: ✅ Available for RAG and multimodal RAG only."* and *"Text Split: ✅ Added for data chunking when you choose an embedding model."* — [Import data wizard](https://learn.microsoft.com/en-us/azure/search/search-import-data-portal), Skills section

Since the wizard uses skillsets with document cracking capabilities, it's **possible** that it supports the same formats as the built-in indexer. But the Import data wizard docs don't publish a separate supported formats list for the Logic Apps connector path.

**What I need from you:**

- Does the Import data wizard (Logic Apps connector path) support the same file formats as the built-in Azure Files indexer?
- If not, which formats are supported? Specifically: PDF, DOCX, XLSX, PPTX, CSV, JSON, HTML, Markdown, plain text?
- Does the wizard's text extraction (via the Logic Apps workflow) use the same document cracking as the built-in indexer, or a different method?
- Should we list supported file types in the tutorial, or just link to the official list?

---

**Question for AI Search team: What is the maximum file size the wizard's Logic Apps workflow can process?**

**Background / How I got here:**

The [Azure File Storage connector](https://learn.microsoft.com/en-us/connectors/azurefile/) documents two separate file size limits:

1. **General limit: 50 MB** — Source: *"Maximum file size (in MB): 50"* — [General Limits](https://learn.microsoft.com/en-us/connectors/azurefile/#general-limits)

2. **Chunking limit: 300 MB** — Source: *"Actions that support chunking feature: Get file content, Get file content using path, Create file, Update file. These actions can be used to handle files up to 300MB. The feature is enabled by default."* — [Actions that support chunking](https://learn.microsoft.com/en-us/connectors/azurefile/#actions-that-support-chunking-feature)

However, the wizard's workflow may not use either of these connector actions. The [Logic Apps connector for AI Search](https://learn.microsoft.com/en-us/azure/search/search-how-to-index-logic-apps) Supported actions section describes the file retrieval step as:

> *"Get the data. An HTTP action that retrieves the uploaded document using the file URL from the trigger output."*

This says **"An HTTP action"** — a raw HTTP action, not the connector's "Get file content" action. If the wizard uses a raw HTTP action, neither the 50 MB nor 300 MB connector limits would apply. Instead, the Logic Apps platform limits would govern:

- **100 MB** without chunking — Source: *"Message size [chunking disabled]: 100 MB"* — [Limits and configuration for Azure Logic Apps](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-limits-and-config), Messages section
- **1 GB** with chunking — Source: *"Message size per action [chunking enabled]: 1 GB"* — Same page, Messages section

Whether chunking is enabled on the wizard's HTTP action depends on its `runtimeConfiguration` setting, which we can't see without inspecting the wizard-generated workflow. From the [chunking docs](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-handle-large-messages):

> *"If an HTTP action doesn't already have chunking enabled, you must set up chunking through the action's `runTimeConfiguration` property."*

**What I need from you:**

- Does the wizard's workflow use a raw HTTP action or the Azure File Storage connector's "Get file content" action to retrieve files?
- What is the effective maximum file size the wizard can process?
- If chunking is not enabled by default on the wizard's HTTP action, can users enable it in the Logic App designer after creation?
