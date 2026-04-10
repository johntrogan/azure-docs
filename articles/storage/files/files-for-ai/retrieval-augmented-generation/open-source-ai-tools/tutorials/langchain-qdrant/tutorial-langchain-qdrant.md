---
title: Build a RAG pipeline using Azure Files with LangChain and Qdrant
description: Learn how to build a retrieval-augmented generation (RAG) pipeline that queries documents stored in Azure Files using LangChain for orchestration and Qdrant as the vector database.
author: ftrichardson1
ms.service: azure-file-storage
ms.topic: tutorial
ms.date: 04/09/2026
ms.author: t-flynnr
ms.custom: devx-track-python
---

# Tutorial: Build a RAG pipeline using Azure Files with LangChain and Qdrant

In this tutorial, you build a retrieval-augmented generation (RAG) pipeline over documents stored in Azure Files. The pipeline uses LangChain for orchestration and Qdrant as the vector database. Qdrant stores all documents in a single collection and uses indexed *payload filtering* to scope queries by department at retrieval time, rather than partitioning data into separate namespaces or indexes.

## Prerequisites

- Complete the [project setup and Azure Files authentication](../../setup.md).

After downloading files from Azure Files (covered in the [setup article](../../setup.md)), split the documents into overlapping chunks. The overlap preserves context at chunk boundaries so the embedding model can capture meaning that spans a split point.

<!-- The rest of the tutorial content should mirror the implementation in your repo. -->
