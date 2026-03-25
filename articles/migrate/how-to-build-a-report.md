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
- After completing this section, you can generate migration and modernization reports.

## Prerequisites 

Before you build a report, ensure the following:

- You’ve created an Azure Migrate project. You can also use an existing project.
After the project is created, the Azure Migrate: Discovery and assessment tool is automatically added. 
- You’ve discovered your IT estate using one of the supported discovery sources, based on your scenario. 
- All discovery errors are resolved before you generate the report.

### Recommendation


We recommend the following actions to improve report accuracy:

- Enrich your data by defining environment, migration intent and application.
- Define the environment for your [workloads to enrich your data](resource-tagging.md).
- Specify the migration [intent and associated applications](define-manage-applications.md).
- Enable application auto‑discovery and review the [discovered applications](resource-tagging.md) for accuracy.

## Build a Report

To build a report, follow these steps:

1. From All projects, **All Projects** select your project. [Picture 1] 
1. Go to **Manage**, and then select **Reports**.

:::image type="content" source="./media/how-to-build-a-report/manage-section.png" alt-text="The screenshot how to access and select reports." lightbox="./media/how-to-build-a-report/manage-section.png":::

1. In **Report name**, enter a name for the report. The report name must be unique within the project.  

:::image type="content" source="./media/how-to-build-a-report/generate-report.png" alt-text="The screenshot how to generate report." lightbox="./media/how-to-build-a-report/generate-report.png":::

1. Select the report type to generate. For more information, see the supported report types.
1. Select the required migration preference. Learn more about migration preferences.
1. Provide the required configuration for generating the report. Learn more about report configuration.
1. Review your selections, and then select **Build report**.

**Report generation time**: 

- Creating a report takes approximately 15 minutes when you select configurations from an existing assessment.
- Creating a report takes approximately 1 hour when you define configurations from scratch.

### Download the report

Follow these steps to download a report:

1. Go to the **Reports** section. All reports created so far are listed here. [Picture 1]
1. For the report you want to download, select **Download**.

:::image type="content" source="./media/how-to-build-a-report/download-report.png" alt-text="The screenshot how to download report." lightbox="./media/how-to-build-a-report/download-report.png":::

1. Select the required files, and then select **Download**.

:::image type="content" source="./media/how-to-build-a-report/report-types.png" alt-text="The screenshot how to select the report types and download." lightbox="./media/how-to-build-a-report/report-types.png":::

## Next steps 

Learn more about how to review the generated reports. 