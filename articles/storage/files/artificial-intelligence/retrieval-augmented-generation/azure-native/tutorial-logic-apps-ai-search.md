---
title: Build a RAG pipeline using Azure Files with Logic Apps and AI Search
description: Learn how to build a no-code retrieval-augmented generation (RAG) pipeline that indexes documents from Azure Files using Azure Logic Apps, Azure OpenAI, and Azure AI Search.
author: ftrichardson1
ms.service: azure-file-storage
ms.topic: tutorial
ms.date: 04/13/2026
ms.author: t-flynnr
---

# Tutorial: Build a RAG pipeline using Azure Files with Logic Apps and AI Search

In this tutorial, you build a no-code retrieval-augmented generation (RAG) pipeline that automatically indexes documents stored in Azure Files. The pipeline uses Azure Logic Apps to orchestrate the workflow, Azure OpenAI to generate vector embeddings, and Azure AI Search to store and query the indexed content. After indexing, you use Azure AI Foundry to ask natural-language questions about your documents.

## Architecture overview

The pipeline works as follows:

1. Azure Logic Apps monitors your Azure file share on a schedule.
1. When documents are found, the Logic App downloads each file, splits it into chunks, and sends each chunk to Azure OpenAI for embedding.
1. The embedded chunks are stored in an Azure AI Search vector index.
1. You query the index through Azure AI Foundry's chat playground, which retrieves relevant chunks and generates grounded answers using a chat completion model.

## Prerequisites

- An Azure subscription. If you don't have one, [create a free account](https://azure.microsoft.com/free/).
- An [Azure file share](/azure/storage/files/storage-files-introduction) containing the documents you want to index and query.
- A resource group for the new resources. You can use the same resource group as your storage account or [create a new one](https://portal.azure.com/#create/Microsoft.ResourceGroup). Use a [supported region](#known-limitations) such as **East US**.
- Permissions to create resources and assign roles in your subscription.

## Step 1: Create an Azure OpenAI resource and deploy models

Azure OpenAI provides the embedding model that converts text into vectors, and the chat completion model that generates answers.

### Create the resource

1. Search for **Azure OpenAI** in the portal.
1. Select **+ Create**.
1. Select your subscription and resource group.
1. Select **East US** as the region.
1. Enter a name, such as `aoai-rag-demo`.
1. For **Pricing tier**, select **Standard S0**.
1. Select **Review + create**, then **Create**.

### Deploy the embedding model

1. Go to your Azure OpenAI resource.
1. In the left menu, select **Model deployments**, then **Manage Deployments** (this opens Azure AI Foundry).
1. Select **+ Create new deployment**.
1. Select **text-embedding-3-large** from the model dropdown.
1. Enter `text-embedding-3-large` as the deployment name.
1. Select **Create**.

> [!IMPORTANT]
> Record the deployment name exactly. You need it when configuring the Logic App. The `text-embedding-3-large` model produces 3,072-dimension vectors—the index schema in a later step must match this value.

### Deploy the chat completion model

1. In Azure AI Foundry, select **+ Create new deployment** again.
1. Select **gpt-4o** (or **gpt-4o-mini**) from the model dropdown.
1. Enter a deployment name, such as `gpt-4o`.
1. Select **Create**.

You need this model later when querying your documents in the Azure AI Foundry chat playground.

### Note your endpoint and key

1. Go back to your Azure OpenAI resource in the Azure portal.
1. In the left menu, under **Resource Management**, select **Keys and Endpoint**.
1. Copy the **Endpoint** URL (looks like `https://<your-resource-name>.openai.azure.com`).
1. Copy **Key1**.

> [!IMPORTANT]
> Store your key securely. Don't share it or commit it to source control.

## Step 2: Create an Azure AI Search resource

Azure AI Search stores the vector index and provides similarity search over your embedded document chunks.

1. Search for **AI Search** in the portal.
1. Select **+ Create**.
1. Select your subscription and resource group.
1. Enter a name, such as `search-rag-demo`. This name must be globally unique.
1. Select **East US** as the region.
1. For **Pricing tier**, select **Basic**.
1. Select **Review + create**, then **Create**.

After deployment, note the following values:

| Value | Location |
| :--- | :--- |
| **Endpoint URL** | **Overview** page (looks like `https://search-rag-demo.search.windows.net`) |
| **Primary admin key** | **Settings** > **Keys** |

## Step 3: Create the vector index in AI Search

> [!IMPORTANT]
> The Logic App template doesn't create this index for you. You must create it manually with the correct vector schema before configuring the workflow. If the schema doesn't match your embedding model, indexing fails silently.

The index requires four fields that the Logic App template expects:

| Field | Purpose |
| :--- | :--- |
| `id` | Unique identifier for each document chunk (generated automatically) |
| `documentName` | Name of the source file, used for citations |
| `content` | Text content of the chunk |
| `embeddings` | Vector representation of the chunk (3,072 dimensions to match `text-embedding-3-large`) |

The index also needs a vector search configuration. The Hierarchical Navigable Small World (HNSW) algorithm organizes vectors for efficient nearest-neighbor search, and cosine similarity measures how closely two vectors match in meaning.

> [!NOTE]
> The `dimensions` value (3,072) must match your embedding model exactly. If you deployed a different model (such as `text-embedding-3-small` with 1,536 dimensions), update this value accordingly. A mismatch causes indexing to fail silently.

### Create the index

1. Go to your AI Search resource.
1. In the left menu, under **Search management**, select **Indexes**.
1. Select **+ Add index**, then select **Add index (JSON)** from the dropdown.
1. Delete the default content and paste the following JSON:

```json
{
  "name": "chunked-index",
  "fields": [
    {
      "name": "id",
      "type": "Edm.String",
      "searchable": true,
      "retrievable": true,
      "key": true
    },
    {
      "name": "documentName",
      "type": "Edm.String",
      "searchable": true,
      "retrievable": true
    },
    {
      "name": "content",
      "type": "Edm.String",
      "searchable": true,
      "retrievable": true
    },
    {
      "name": "embeddings",
      "type": "Collection(Edm.Single)",
      "searchable": true,
      "filterable": false,
      "retrievable": true,
      "dimensions": 3072,
      "vectorSearchProfile": "vector-profile"
    }
  ],
  "vectorSearch": {
    "algorithms": [
      {
        "name": "vector-config",
        "kind": "hnsw",
        "hnswParameters": {
          "metric": "cosine",
          "m": 4,
          "efConstruction": 400,
          "efSearch": 500
        }
      }
    ],
    "profiles": [
      {
        "name": "vector-profile",
        "algorithm": "vector-config"
      }
    ]
  }
}
```

5. Select **Save**.

## Step 4: Create the Logic App

Azure Logic Apps orchestrates the entire pipeline. It monitors your file share, chunks documents, generates embeddings, and writes to the search index—all without code.

1. Search for **Logic App** in the portal.
1. Select **+ Create**.
1. For plan type, select **Standard** with **Workflow Service Plan (WS1)**.
1. Select your subscription and resource group.
1. Enter a name, such as `logic-app-rag-demo`.
1. Select **East US** as the region.
1. Select **Review + create**, then **Create**.

> [!NOTE]
> You must use a Standard Logic App (Workflow Service Plan), not Consumption. The RAG template is only available in the Standard plan.

## Step 5: Configure the Logic App workflow from template

Instead of building the workflow from scratch, use the pre-built template that automates the entire RAG indexing pipeline.

1. Go to your Logic App resource.
1. In the left menu, select **Workflows**.
1. Select **+ Add**, then select **Add from Template**.
1. Search for `azure ai search` and select the template named **Azure Files: Ingest and index documents at a schedule using Azure OpenAI and Azure AI Search - RAG pattern**.
1. Select **Use this template**.
1. Enter a workflow name, such as `index-azure-files-rag`.
1. Select **Next**.

### Configure connections

The template requires three connections:

#### Azure Files connection

| Setting | Value |
| :--- | :--- |
| **Storage account URI** | Found in your storage account under **Settings** > **Endpoints** > **File service** (looks like `https://<your-storage-account>.file.core.windows.net`) |
| **Connection string** | Found in your storage account under **Security + networking** > **Access keys** > **Connection string** |

#### Azure AI Search connection

| Setting | Value |
| :--- | :--- |
| **Endpoint URL** | From your AI Search resource **Overview** page |
| **Admin key** | From AI Search **Settings** > **Keys** > **Primary admin key** |

#### Azure OpenAI connection

| Setting | Value |
| :--- | :--- |
| **Endpoint URL** | From your Azure OpenAI resource **Keys and Endpoint** page |
| **Key** | **Key1** from the same page |

Select **Connect** for each connection, then select **Next**.

### Configure indexing parameters

| Parameter | Value |
| :--- | :--- |
| **AI Search index name** | `chunked-index` (must match the index you created earlier) |
| **OpenAI text embedding deployment identifier** | `text-embedding-3-large` (must match the deployment name) |
| **Azure Files storage folder name** | The name of your existing Azure file share |

Select **Next**, then **Create**.

## Step 6: Run and validate the workflow

1. Select **Go to My Workflow** after creation.
1. The workflow runs on a schedule. To trigger it immediately, select **Run Trigger** > **Run**.
1. Check the **Run history** to verify the run succeeded (green checkmarks indicate success).
1. If a run fails, select it to see which step failed. Common causes include mismatched index names, incorrect deployment names, or expired keys.

### Verify the index

1. Go to your AI Search resource.
1. Under **Search management**, select **Indexes**.
1. Select **chunked-index**.
1. Verify that the document count is greater than zero.
1. Select **Search explorer** and run an empty search (`*`) to see the indexed chunks.

## Step 7: Chat with your documents using Azure AI Foundry

1. Go to [Azure AI Foundry](https://ai.azure.com).
1. Create or open a project using your subscription and resource group.
1. In the left menu, select **Playgrounds** > **Chat**.
1. Select **Add your data**.
1. Select **Azure AI Search** as the data source.
1. Enter your AI Search endpoint, admin key, and select the `chunked-index` index.
1. Complete the setup wizard.
1. Type a question about your documents. The system retrieves relevant chunks from the index and uses your `gpt-4o` deployment to generate a grounded answer with citations.

> [!NOTE]
> If you created the index without a query-time vectorizer, use the default setup in Foundry. If you added a vectorizer to the index, remove the vector option from the Foundry configuration to avoid double vectorization.

## Cost estimate

The following table summarizes the approximate monthly costs for the resources created in this tutorial. Costs vary by region and usage.

| Resource | SKU | Approximate monthly cost |
| :--- | :--- | :--- |
| Azure OpenAI | Standard S0 | Per-token pricing (~$0.13 per 1M tokens for `text-embedding-3-large`). Negligible for small document sets. |
| Azure AI Search | Basic | ~$75 |
| Logic App | Standard WS1 | ~$150 |
| Storage account (Azure Files) | Standard LRS | Already exists (pennies per GB for small file shares) |

The biggest ongoing costs are **AI Search Basic** and **Logic App Standard WS1**. For a short-lived demo, you can delete the resource group immediately after testing to avoid charges. For pricing details, see [Azure AI Search pricing](https://azure.microsoft.com/pricing/details/search/), [Azure Logic Apps pricing](https://azure.microsoft.com/pricing/details/logic-apps/), and [Azure OpenAI pricing](https://azure.microsoft.com/pricing/details/cognitive-services/openai-service/).

## Clean up resources

To avoid ongoing charges, delete the resource group:

```azurecli
az group delete --name rg-logic-apps-rag-demo --yes --no-wait
```

> [!NOTE]
> Your Azure file share may be shared infrastructure—confirm with your administrator before deleting. The AI Search Basic tier (~$75/month) and Logic App Standard WS1 (~$150/month) are the most significant ongoing costs.

## Known limitations

- The Logic App template doesn't create the AI Search index. You must create it manually with the correct vector schema.
- The `dimensions` value in the index must match your embedding model exactly (3,072 for `text-embedding-3-large`).
- The Logic App must be Standard (Workflow Service Plan), not Consumption.
- The workflow indexes files found at run time. Files added after the last run are picked up on the next scheduled execution.
- The template is available only in regions that support all three services (Logic Apps Standard, Azure OpenAI, and Azure AI Search).
- The template uses a fixed chunking strategy. You can't customize chunk size or overlap without modifying the workflow definition.

## Related content

- [What is retrieval-augmented generation?](../overview.md)
- [Azure Logic Apps documentation](/azure/logic-apps/logic-apps-overview)
- [Azure AI Search documentation](/azure/search/search-what-is-azure-search)
- [Azure OpenAI documentation](/azure/ai-services/openai/overview)
- [Azure AI Foundry documentation](/azure/ai-studio/what-is-ai-studio)
