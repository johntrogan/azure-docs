---
title: Build an Azure Migrate Report 
description: Build Azure Migrate reports to analyze discovered on-premises servers and workloads and generate insights for migration planning.
author: habibaum
ms.author: v-uhabiba
ms.topic: how-to
ms.service: azure-migrate
ms.date: 03/24/2026
monikerRange:
# Customer intent: As an IT administrator managing migration resources, I want to tag workloads with relevant attributes, so that I can enhance resource organization and visibility during the migration process.
---

# Build a report (preview) 

This article explains how to build a report (preview) for on‑premises servers and workloads by using Azure Migrate. After completing this article, you’ll be able to generate a report by selecting the appropriate report type, migration preferences, and configuration options in an Azure Migrate project.

In this article, you’ll learn how to:

- Create a report in Azure Migrate.
- Select the appropriate report type, migration preferences, and configuration options.
- Generate the report to review insights about your discovered servers and workloads.

## Prerequisites 

Before you build a report, ensure the following:

- You’ve created an Azure Migrate project. You can also use an existing project.
After the project is created, the Azure Migrate: Discovery and assessment tool is automatically added. 
- You’ve discovered your IT estate using one of the supported discovery sources, based on your scenario. 
- All discovery errors are resolved before you generate the report.

### Recommendation


We recommend the following actions to improve report accuracy:

- Define the environment for your [workloads to enrich your data](resource-tagging.md).
- Specify the migration [intent and associated applications](define-manage-applications.md).
- Enable application auto‑discovery and review the [discovered applications](resource-tagging.md) for accuracy.

## Build a Report

To build a report, follow these steps:

1. From All projects, **All Projects** select your project. [Picture 1] 
1. Go to **Manage**, and then select **Reports**. [Picture 2] 
1. In **Report name**, enter a name for the report. The report name must be unique within the project.  [Picture 3] 
1. Select the report type to generate. For more information, see the supported report types.
1. Select the required migration preference. Learn more about migration preferences.
1. Provide the required configuration for generating the report. Learn more about report configuration.
1. Review your selections, and then select **Build report**.

## Next steps 

Learn more about how to review the generated reports. 