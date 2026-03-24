---
title: Azure Migrate Reports Overview 
description: Learn how Azure Migrate Reports help you create decision‑ready reports with insights into security, readiness, cost, ROI, and migration strategies for Azure planning
author: habibaum
ms.author: v-uhabiba
ms.topic: how-to
ms.service: azure-migrate
ms.date: 03/24/2026
monikerRange:
# Customer intent: As an IT administrator managing migration resources, I want to tag workloads with relevant attributes, so that I can enhance resource organization and visibility during the migration process.
---

# Overview of reports

Azure Migrate reports deliver decision-ready insights by consolidating key information across workloads, security, readiness, and costs. They help stakeholders evaluate Azure targets, business value, and migration strategies with a high-level view of the discovered estate. You can create reports after completing discovery, defining applications, and enriching the inventory with environment and migration intent details.
 
## What are reports? 

Reports in Azure Migrate help you generate decision-ready executive reports for the modernization and migration of your discovered estate. These reports provide a high-level view of your environment, help evaluate migration and modernization options, and support communication of the business value of moving to Azure. 

Reports are designed to summarize your estate and highlight migration opportunities by bringing together key insights across applications, infrastructure, and data. The outputs are intended to support stakeholder discussions, planning activities, and informed decision-making. 

Depending on the selected report type and scenario, reports can summarize insights such as:
- Workload and application summary across your discovered estate, including web apps, databases, and file servers.
- Security insights and vulnerability summary for discovered workloads.
- Utilization summary for workloads across the estate.
- Software inventory summary, including the types of software supporting your applications.
- Total cost of ownership (TCO) and return on investment (ROI) considerations for migration and modernization scenarios.
- Value realization on Azure, including potential savings through Azure benefits, management solutions, and pricing options.
- Readiness, target recommendations, and Azure cost insights for workloads based on the selected migration preference. 
- High-level migration wave plan to support phased migration planning.
- The report generates exportable, read-only outputs in PowerPoint and Excel formats.
 
## Types of reports 

Azure Migrate provides the following reports to help you assess your environment and plan your migration and modernization journey.

### Azure modernization and migration report

The Azure modernization and migration report provides a consolidated view of migration and modernization opportunities across applications, infrastructure, data, web apps, file share servers, and software.

This report helps you:

- Understand your current environment and infrastructure
- Assess security and vulnerability posture
- Identify software that supports your applications
- Determine recommended Azure target services, readiness, SKUs, and estimated Azure costs for your selected migration strategy
- Evaluate return on investment (ROI), business value, and potential savings to support stakeholder discussions and executive decision-making

[Learn more](#migration-preferences-in-azure-migrate-reports) about the migration strategies supported by the Azure modernization and migration report.

### Security insights report

The Security insights report provides an overview of the security assessment for infrastructure, software, web apps, and databases discovered in your datacenter.

This report helps you:

- Understand your overall security posture
- Identify vulnerabilities in your on-premises environment

## Migration preferences in Azure Migrate reports

When you build a report, you can choose from two migration preferences:

- If you want a slightly more formal Learn tone (optional variant):
- When generating a report, you can select one of two migration preferences.


| **Migration Strategy** |  **Details**  | **Assessment insights**|
| --- | --- | --- |
| **Modernize (Platform as a Service)** | You can get a PaaS preferred recommendation that means, the logic identifies workloads best fit for PaaS targets.<br><br>General servers are recommended with a quick lift and shift recommendation to Azure IaaS. | For SQL Servers, sizing and cost comes from the Recommended report with optimization strategy - *Modernize to PaaS* from Azure SQL assessment.<br><br>For web apps, sizing and cost comes from Azure App Service and Azure Kubernetes Service assessments, with a preference to App Service. For general servers, sizing and cost comes from Azure VM assessment.<br><br>All of these recommendations are aggregated using the heterogenous assessments. |
| **Migrate (Infrastructure as a Service)** | You can get a quick lift and shift recommendation to Azure IaaS (Azure VM or Azure AVS). | **When Lift and Shift to Azure VM is selected:**<br>For SQL Servers, sizing and cost comes from the *Instance to SQL Server on Azure VM report*.<br><br>For general servers and servers hosting web apps, sizing and cost comes from Azure VM assessment.<br><br>All of these recommendations are aggregated using the heterogenous assessments.<br><br>**When Lift and Shift to Azure VMware Solution is selected:**<br>For all general servers, hosted on VMware, sizing and cost comes from Azure Vmware Solution (AVS) assessment.<br><br>All of these recommendations are aggregated using the heterogenous assessments. |

Reports surface Azure recommendations from heterogeneous assessments and provide direct access to those assessments. To review sizing, readiness, and Azure cost estimates in detail, open the relevant assessment for the selected applications or workloads.

### Discovery sources

You can create a report using either of the following discovery methods:

- **Azure Migrate appliance or Azure Migrate collector–based discovery**: Provides the most accurate inventory, metadata, and performance data.
- **CSV import**: Provides a quick estimate when inventory data is available in CSV format.

### Report configuration

You can configure report generation in one of the following ways:

- **Define configuration**: Specify the scope, settings, and configuration to be used when generating the report.
- **Use configuration from an existing assessment**: Select an existing assessment and reuse its scope, settings, and configuration to generate the report.


## Next steps 

- Learn more about [how to build a report](how-to-build-a-report.md). 