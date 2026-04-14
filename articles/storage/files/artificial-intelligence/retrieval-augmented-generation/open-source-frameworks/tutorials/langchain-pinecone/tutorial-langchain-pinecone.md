---
title: Build a RAG pipeline using Azure Files with LangChain and Pinecone
description: Learn how to build a retrieval-augmented generation (RAG) pipeline that queries documents stored in Azure Files using LangChain for orchestration and Pinecone as the vector database.
author: ftrichardson1
ms.service: azure-file-storage
ms.topic: tutorial
ms.date: 04/09/2026
ms.author: t-flynnr
ms.custom: devx-track-python
---

# Tutorial: Build a RAG pipeline using Azure Files with LangChain and Pinecone

In this tutorial, you build a retrieval-augmented generation (RAG) pipeline over documents stored in Azure Files. The pipeline uses LangChain for orchestration and Pinecone as the vector database.

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
pip install langchain langchain-openai langchain-pinecone pinecone
```

## Step 1: Parse and chunk documents

After downloading files from Azure Files (covered in the [setup article](../../setup.md)), split the documents into overlapping chunks. The overlap preserves context at chunk boundaries so the embedding model can capture meaning that spans a split point.

```python
def chunk_documents(documents):
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=CHUNK_SIZE,
        chunk_overlap=CHUNK_OVERLAP,
    )
    return splitter.split_documents(documents)
```

`RecursiveCharacterTextSplitter` splits each document into chunks of `CHUNK_SIZE` characters with `CHUNK_OVERLAP` characters of overlap between adjacent chunks. All original metadata (such as file path) is automatically copied to each child chunk.

## Step 2: Create embeddings and index into Pinecone

Connect to your Pinecone index, create an embedding model using Azure OpenAI, and upsert the vectors.

```python
def embed_and_index(chunks):
    pc = Pinecone(api_key=PINECONE_API_KEY)

    if not pc.has_index(PINECONE_INDEX_NAME):
        pc.create_index(
            name=PINECONE_INDEX_NAME,
            dimension=EMBEDDING_DIMENSIONS,
            metric="cosine",
            spec=ServerlessSpec(cloud="azure", region="eastus2"),
        )

    embeddings = AzureOpenAIEmbeddings(
        azure_endpoint=OPENAI_ENDPOINT,
        azure_deployment=OPENAI_EMBEDDING_DEPLOYMENT,
        azure_ad_token_provider=TOKEN_PROVIDER,
        dimensions=EMBEDDING_DIMENSIONS,
    )

    return PineconeVectorStore.from_documents(
        documents=chunks,
        embedding=embeddings,
        index_name=PINECONE_INDEX_NAME,
    )
```

This function:

1. **Creates the index if needed**—`has_index()` checks whether the Pinecone index exists. If not, `create_index()` creates a serverless index with the correct dimension and cosine distance metric.
2. **Creates the embedding model**—`AzureOpenAIEmbeddings` authenticates to Azure OpenAI using Entra ID tokens (via `azure_ad_token_provider`), not API keys.
3. **Embeds and upserts**—`PineconeVectorStore.from_documents()` batches the embedding API calls and upserts the resulting vectors into the Pinecone index.

## Step 3: Build the retrieval chain

Build a LangChain Expression Language (LCEL) chain that retrieves relevant chunks from Pinecone and generates an answer using Azure OpenAI.

```python
def build_qa_chain(vector_store):
    llm = AzureChatOpenAI(
        azure_endpoint=OPENAI_ENDPOINT,
        azure_deployment=OPENAI_CHAT_DEPLOYMENT,
        azure_ad_token_provider=TOKEN_PROVIDER,
        api_version="2024-12-01-preview",
    )
    retriever = vector_store.as_retriever(
        search_type="similarity",
        search_kwargs={"k": 5},
    )

    prompt = PromptTemplate.from_template(
        "Answer the question based on the context below. "
        "Be specific and cite the source file name in brackets for each fact.\n\n"
        "Context:\n{context}\n\n"
        "Question: {question}\n\nAnswer:"
    )

    def format_docs(docs):
        return "\n\n".join(
            f"[{d.metadata.get('azure_file_path', '')}]\n{d.page_content}"
            for d in docs
        )

    return (
        {"context": retriever | format_docs, "question": RunnablePassthrough()}
        | prompt | llm | StrOutputParser()
    )
```

The LCEL chain has four stages:

1. **Retrieve**—The user's question is vectorized and the top *k* most similar chunks are retrieved from Pinecone.
2. **Format**—The retrieved chunks are joined into a single context string, with each chunk prefixed by its source file path for citation.
3. **Prompt**—The context and question are injected into a template that instructs the model to answer and cite sources.
4. **Generate**—`AzureChatOpenAI` produces an answer and `StrOutputParser()` extracts the text.

## Step 4: Run the pipeline

Run the pipeline script:

```bash
python langchain-pinecone.py
```

> [!NOTE]
> The code snippets in Steps 1–3 show individual functions for clarity. The full runnable script that ties these functions together (including file downloading, document parsing, and the interactive Q&A loop) is available in the [azure-files-langchain-pinecone](https://github.com/ftrichardson1/azure-files-langchain-pinecone) GitHub repository.

The script scans the Azure file share, downloads and parses documents, chunks them, indexes them into Pinecone, and starts an interactive query session. Type a question and press Enter. Type `quit` to exit.

## Clean up resources

To delete the Azure resources created for this tutorial:

```bash
az group delete --name rg-rag-demo --yes --no-wait
```

> [!NOTE]
> Your Azure file share may be shared infrastructure—confirm with your administrator before deleting. To remove your Pinecone index, delete it from the [Pinecone console](https://app.pinecone.io/).

## Next steps

- [LangChain documentation](https://python.langchain.com/docs/introduction/)
- [Pinecone documentation](https://docs.pinecone.io/)
- [LangChain Pinecone integration](https://python.langchain.com/docs/integrations/vectorstores/pinecone/)
- [Azure OpenAI documentation](/azure/ai-services/openai/)