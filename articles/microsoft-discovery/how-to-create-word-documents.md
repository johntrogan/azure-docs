---
title: Create Word documents with a custom tool in Microsoft Discovery
description: Learn how to use a custom Word document tool to create and edit .docx files in Microsoft Discovery investigations, enabling agents to produce structured reports and documents.
author: hectoralinares
ms.author: hectorl
ms.service: azure
ms.topic: how-to
ms.date: 04/02/2026

#CustomerIntent: As a researcher or scientist, I want to produce Word documents from my investigations so that I can share structured reports with collaborators who need formatted documents.
---

# Create Word documents with a custom tool

The built-in file tools in Microsoft Discovery work with text-based formats like Markdown, CSV, and JSON. To produce binary file formats such as Word documents (.docx), you use a custom tool that runs on a [supercomputer](how-to-manage-supercomputers.md). This guide shows how to set up and use a Word document tool, and explains the pattern you can apply to other binary formats.

## How the tool works

The Word document tool is a containerized Python application that uses the `python-docx` library to create and manipulate .docx files. Agents interact with the document through structured actions rather than reading or writing binary data directly.

The tool exposes these actions:

| Action | What it does |
|--------|-------------|
| **initializeDocument** | Creates a new empty .docx file |
| **addTitle** | Adds a title (heading level 0) |
| **addHeading** | Adds a heading at a specified level (1-9) |
| **addText** | Adds a paragraph of text |
| **addPageBreak** | Inserts a page break |
| **initializeTable** | Creates a table with column headers |
| **insertRow** | Adds a row to an existing table |
| **viewDocument** | Returns the full document content as plain text |
| **getContent** | Returns structured content (paragraphs with styles, table data) as JSON |
| **getSections** | Returns a list of all non-empty paragraphs |

Each action that modifies the document saves it and uploads it to blob storage. The agent builds the document incrementally across multiple action calls.

## Prerequisites

Before using the Word document tool, you need:

- A [supercomputer with a nodepool](how-to-manage-supercomputers.md) configured in your workspace
- An [Azure Container Registry](concept-azure-container-registry.md) accessible from your workspace
- The Word tool container image built and pushed to your registry
- The tool registered as a Discovery tool resource
- An agent configured with the tool assigned

## Build the container image

The Word tool container requires three Python packages:

- `python-docx` for creating and editing Word documents
- `azure-storage-blob` for uploading documents to blob storage
- `azure-identity` for authenticating with Azure services

The container runs on the supercomputer using the managed identity configured on the nodepool, which provides access to the blob storage account.

## Register the tool

Register the Word tool as a Discovery tool resource using the ARM API. The tool definition specifies the container image, compute requirements, and the actions that agents can call. Each action maps to a command-line invocation of the tool's Python script.

Key configuration:

- **Container image**: Point to your ACR registry where the Word tool image is stored
- **Compute requirements**: The Word tool is lightweight (1-2 CPU, 2-4 GB RAM, no GPU required)
- **Pool type**: Use a static pool for consistent availability, or a dynamic pool if you use the tool infrequently
- **Environment variables**: Configure the storage account and container name where documents are stored

## Configure an agent with the tool

Create or update an agent to include the Word tool. When the tool is assigned to an agent, the agent gains access to all of the tool's actions (initializeDocument, addTitle, addHeading, and so on) alongside the standard built-in tools.

Give the agent instructions that explain how to use the Word tool effectively. For example:

> "You are a scientific report writer. Use the Word document tool to create structured .docx reports. Always initialize a document first, then add a title, then organize content with headings and paragraphs. Use tables for structured data. Call viewDocument at the end to verify the content."

## Create a document in an investigation

### Task description

Write task descriptions that tell the agent what document to create and what content to include:

> "Create a Word document called protein_analysis_report. Add a title 'Protein Analysis Results'. Add sections for Methodology, Results, and Discussion. In the Methodology section, describe the BLAST search parameters used. In the Results section, include a table with columns for Protein Name, Percent Identity, and E-Value with the top 5 hits. In the Discussion section, summarize the significance of the findings."

### Validation requirements

Since the built-in preview tools can't read .docx files, write validation requirements based on what the agent reports in the task result text:

- "The agent must confirm the document was created and uploaded successfully"
- "The response must include the output of viewDocument showing all three sections"
- "The viewDocument output must include a table with at least 5 rows of protein data"

The tool's **viewDocument** action extracts text from the .docx and returns it to the agent. When you require the agent to include the viewDocument output in its response, the validation agent can verify the document content through the task result text.

## Build multi-section reports across tasks

For longer reports, use multiple child tasks to write sections in parallel, then a parent task to merge them:

```
Root Task: "Compile the final analysis report"
  Depends on: Chemistry Section, Biology Section, Environment Section

  Chemistry Section (child):
    "Use the Word tool to create a document called chem_section.
     Add a heading 'Chemistry Analysis' and write the findings."

  Biology Section (child):
    "Use the Word tool to create a document called bio_section.
     Add a heading 'Biology Analysis' and write the findings."

  Environment Section (child):
    "Use the Word tool to create a document called env_section.
     Add a heading 'Environmental Analysis' and write the findings."

  Root Task:
    "Use the Word tool viewDocument action to read each child document
     (chem_section, bio_section, env_section). Create a new document
     called final_report. Add a title and combine all sections with
     an introduction and conclusion."
```

> [!NOTE]
> Each child task's agent creates its document independently through the tool. The root task's agent reads the child documents using the tool's viewDocument action and creates the combined report. This works because the tool stores documents in a shared blob container accessible to all agents in the workspace.

## Apply this pattern to other binary formats

The Word tool demonstrates a pattern that works for any binary format:

1. **Find a Python library** that handles the format (python-docx for Word, PyMuPDF for PDF, openpyxl for Excel, Pillow for images)
2. **Containerize it** with azure-storage-blob and azure-identity for storage access
3. **Expose actions** that let agents create, read, and modify the content through text-based inputs and outputs
4. **Include a view/read action** that extracts text content so agents can reason about the file without needing the platform's built-in preview

This approach lets agents work with any file format while keeping the interaction text-based.

## Current limitations

| Limitation | Description |
|-----------|-------------|
| **Documents stored separately** | The Word tool stores .docx files in its own blob container, not as Discovery storage assets. Documents don't appear in the task's `storageAssetIds` and aren't visible through the built-in GetResourceContext tool. The team is evaluating integration with the Discovery storage asset system. |
| **No built-in preview** | The platform's built-in file preview can't read .docx files. Use the tool's viewDocument action instead. |
| **No cross-tool file access** | An agent using a different tool can't read a .docx created by the Word tool through built-in tools. The Word tool's viewDocument action is the only way to read the content. |
| **Cold start delay** | The first tool call in a session requires the supercomputer to start a container, which can take 2-4 minutes. Subsequent calls to the same tool are faster. |

## Related content

- [Files and storage assets](concept-files-storage-assets.md)
- [Agent types](concept-discovery-agent-types.md)
- [Manage Supercomputer and Nodepools](how-to-manage-supercomputers.md)
- [Azure Container Registry](concept-azure-container-registry.md)
- [Task addition and execution](how-to-task-addition-execution.md)
