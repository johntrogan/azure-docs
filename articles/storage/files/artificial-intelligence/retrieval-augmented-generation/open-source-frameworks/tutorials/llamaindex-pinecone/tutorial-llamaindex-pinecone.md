---
title: Build a RAG pipeline using Azure Files with LlamaIndex and Pinecone
description: Learn how to build a retrieval-augmented generation (RAG) pipeline that queries documents stored in Azure Files using LlamaIndex for orchestration and Pinecone as the vector database.
author: ftrichardson1
ms.service: azure-file-storage
ms.topic: tutorial
ms.date: 04/09/2026
ms.author: t-flynnr
ms.custom: devx-track-python
---

# Tutorial: Build a RAG pipeline using Azure Files with LlamaIndex and Pinecone

In this tutorial, you build a retrieval-augmented generation (RAG) pipeline over documents stored in Azure Files. The pipeline uses LlamaIndex for orchestration and Pinecone as the vector database.

## Prerequisites

- Complete the [project setup and Azure Files authentication](../../setup.md).
- An [Azure OpenAI](/azure/ai-services/openai/how-to/create-resource) resource with the following deployments:
  - A text embedding model (for example, `text-embedding-3-small`)
  - A chat completion model (for example, `gpt-4o`)
- A [Pinecone account](https://www.pinecone.io/) (the free tier is sufficient). You need an API key and an index name from the [Pinecone console](https://app.pinecone.io/).

> [!IMPORTANT]
> Store your Pinecone API key securely. Do not commit API keys to source control.

Set the following environment variables in your `.env` file:

```text
PINECONE_API_KEY=<your-pinecone-api-key>
PINECONE_INDEX_NAME=<your-pinecone-index-name>
```

| Variable | Description |
| :--- | :--- |
| `PINECONE_API_KEY` | Your Pinecone API key from the [Pinecone console](https://app.pinecone.io/) |
| `PINECONE_INDEX_NAME` | The name of your Pinecone index |

## Install dependencies

Install the required packages for this tutorial:

```bash
pip install llama-index llama-index-embeddings-azure-openai llama-index-llms-azure-openai llama-index-vector-stores-pinecone llama-index-readers-file pinecone
```

## Step 1: Parse and chunk documents

After downloading files from Azure Files (covered in the [setup article](../../setup.md)), split the documents into overlapping nodes. LlamaIndex calls chunks *nodes* — each node carries metadata and relationship information from its parent document.

```python
from llama_index.core.node_parser import SentenceSplitter

def chunk_documents(documents):
    splitter = SentenceSplitter(
        chunk_size=CHUNK_SIZE,
        chunk_overlap=CHUNK_OVERLAP,
    )
    return splitter.get_nodes_from_documents(documents)
```

`SentenceSplitter` splits each document into nodes of `CHUNK_SIZE` characters with `CHUNK_OVERLAP` characters of overlap between adjacent nodes. Unlike a character-only splitter, it respects sentence boundaries where possible. All original metadata (such as file path) is automatically copied to each child node.

## Step 2: Create embeddings and index into Pinecone

Connect to Pinecone, embed the nodes with Azure OpenAI, and upsert the vectors into a Pinecone index.

```python
def embed_and_index(nodes):
    pc = Pinecone(api_key=PINECONE_API_KEY)

    if not pc.has_index(PINECONE_INDEX_NAME):
        pc.create_index(
            name=PINECONE_INDEX_NAME,
            dimension=EMBEDDING_DIMENSIONS,
            metric="cosine",
            spec=ServerlessSpec(cloud="azure", region="eastus2"),
        )

    pinecone_index = pc.Index(PINECONE_INDEX_NAME)

    embed_model = AzureOpenAIEmbedding(
        azure_endpoint=OPENAI_ENDPOINT,
        azure_deployment=OPENAI_EMBEDDING_DEPLOYMENT,
        azure_ad_token_provider=TOKEN_PROVIDER,
        use_azure_ad=True,
        model="text-embedding-3-small",
        dimensions=EMBEDDING_DIMENSIONS,
        api_version="2024-06-01",
    )

    vector_store = PineconeVectorStore(pinecone_index=pinecone_index)
    storage_context = StorageContext.from_defaults(vector_store=vector_store)

    return VectorStoreIndex(
        nodes=nodes,
        storage_context=storage_context,
        embed_model=embed_model,
    )
```

This function:

1. **Creates the Pinecone index**—`Pinecone.has_index()` checks whether the index exists, and `create_index()` creates it if needed with the correct dimension and metric.
2. **Creates the embedding model**—`AzureOpenAIEmbedding` authenticates to Azure OpenAI using Entra ID tokens (via `azure_ad_token_provider` and `use_azure_ad=True`), not API keys.
3. **Builds a `VectorStoreIndex`**—LlamaIndex separates the vector store, storage context, and index into distinct layers. `PineconeVectorStore` wraps the Pinecone index. `StorageContext.from_defaults()` connects that vector store to LlamaIndex's storage layer. `VectorStoreIndex()` then embeds all nodes and upserts them into Pinecone in one step.

## Step 3: Build the query engine

Build a LlamaIndex query engine that retrieves relevant nodes from Pinecone and generates an answer using Azure OpenAI.

```python
def build_query_engine(index):
    llm = AzureOpenAI(
        engine=OPENAI_CHAT_DEPLOYMENT,
        model="gpt-4o-mini",
        azure_endpoint=OPENAI_ENDPOINT,
        azure_ad_token_provider=TOKEN_PROVIDER,
        use_azure_ad=True,
        api_version="2024-06-01",
    )
    return index.as_query_engine(
        llm=llm,
        similarity_top_k=5,
        text_qa_template=PromptTemplate(
            "Answer the question based on the context below. "
            "Be specific and cite the source file name in brackets for each fact.\n\n"
            "Context:\n{context_str}\n\n"
            "Question: {query_str}\n\nAnswer:"
        ),
    )
```

The query engine handles retrieval and response synthesis in a single abstraction:

1. **Creates the LLM**—`AzureOpenAI` uses `engine` as the deployment name and authenticates via Entra ID with `use_azure_ad=True`.
2. **Builds the query engine**—`index.as_query_engine()` creates a retrieve-synthesize pipeline that vectorizes the user's question, retrieves the top *k* nodes from Pinecone via cosine similarity, and synthesizes a response. A custom `PromptTemplate` instructs the model to cite the source file name for each fact.

## Step 4: Run the pipeline

Run the pipeline script:

```bash
python llamaindex-pinecone.py
```

The script scans the Azure file share, downloads and parses documents, chunks them, indexes them into Pinecone, and starts an interactive query session. Ask questions about your documents and type `quit` to exit.

> [!NOTE]
> The full runnable script is available in the [azure-files-llamaindex-pinecone](https://github.com/ftrichardson1/azure-files-llamaindex-pinecone) GitHub repository.

## Clean up resources

To delete the Azure resources created for this tutorial:

```bash
az group delete --name rg-rag-demo --yes --no-wait
```

> [!NOTE]
> Your Azure file share may be shared infrastructure—confirm with your administrator before deleting. To remove your Pinecone index, delete it from the [Pinecone console](https://app.pinecone.io/).

## Next steps

- [LlamaIndex documentation](https://docs.llamaindex.ai/)
- [Pinecone documentation](https://docs.pinecone.io/)
- [LlamaIndex Pinecone integration](https://developers.llamaindex.ai/python/examples/vector_stores/pineconeindexdemo)
- [Azure OpenAI documentation](/azure/ai-services/openai/)
