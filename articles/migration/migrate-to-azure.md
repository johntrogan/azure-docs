---
title: Migrate Workloads to Azure
description: Learn about migration resources that might help you transition workloads from AWS, GCP and on-premises to Azure.
author: reginahack
ms.author: rhackenberg
ms.date: 03/24/2025
ms.topic: concept-article
ms.service: azure
ms.custom: migration-hub
ms.collection:
 - migration
 - aws-to-azure
 - gcp-to-azure
 - onprem-to-azure

---

# Migrate workloads to Azure

The Azure Migration Hub provides prescriptive, opinionated guidance to help workload teams plan and implement their migration to Azure. It covers migrations from on‑premises environments and cloud platforms such as Amazon Web Services (AWS) and Google Cloud Platform (GCP).

> [!IMPORTANT]
> This content covers single-workload migrations. It doesn't cover full datacenter migrations, region relocations, or hybrid workloads that run concurrently on multiple clouds.

Migration to Azure touches networking, identity, databases, compute, storage, and custom glue your team built over the years. The guidance for all of this lives across different articles and guides.

This article helps you figure out which guidance applies to you. It asks where your workload is today, then sends you to the right guide. It also covers general migration terminology and strategies.

## Who should read this article

Read this article if you're a workload architect or engineer who was told "we're moving to Azure" and you need to know where to start. You might be coming from AWS, Google Cloud, or an on-premises datacenter. You might not know yet whether you should rehost, replatform, or refactor.

- **Workload architects** who redesign architecture aspects and validate the overall design to meet business requirements on Azure. Architects address gaps by considering the workload's specific characteristics and business constraints.
- **Workload team members** who need to understand how their responsibilities change during and after migration. For example, database administrators who manage scripts and perform daily backups on Amazon Relational Database Service must adapt to performing the same tasks on Azure SQL Database.

This article helps you skip the “where do I start?” phase and jump straight to the migration guide that fits your scenario.

## Migration strategies

The following high-level overview describes different migration strategies. Each strategy involves different levels of risk, effort, and reward. The right choice depends on your workload's complexity, your timeline, and how much you want to change during the move.

- **Rehost (lift and shift):** Move the workload as-is to Azure infrastructure without modifying code. This approach is fast and low risk. It works well for straightforward workloads where the priority is getting off the source platform quickly. For example, you might migrate a web application running on a Windows Server VM to an Azure Virtual Machine. You get the benefits of Azure infrastructure without changing the workload's architecture or code. You change where it runs, not how it runs.
- **Replatform (lift, adjust, and shift):** Make minimal changes to take advantage of Azure platform services. For example, you migrate a SQL Server database to Azure SQL Managed Instance. You get operational benefits without a full rewrite.
- **Refactor:** Restructure existing code to improve performance, scalability, or maintainability without changing the workload's external behavior. For example, you refactor a monolithic .NET application to run on Azure App Service by replacing Windows-specific file path handling, session state handling, and local disk logging. Refactoring takes more effort up front, but lowers operational overhead in the long run.
- **Rearchitect:** Redesign the workload to take full advantage of Azure's capabilities. For example, you redesign a web application to use Azure Functions and Azure Cosmos DB instead of VMs and SQL Server. This approach requires significant code changes but can yield the greatest benefits in scalability, performance, and cost optimization.
- **Retire:** Decommission the workload when it's no longer needed. This strategy is valid if the workload is obsolete, redundant, or can be replaced by a SaaS solution. For example, you might retire an on-premises file server if all its data is migrated to Azure Files and users are trained to access files from there.
- **Replace:** Adopt a turnkey cloud service instead of migrating your existing implementation. Consider this option when a SaaS solution meets your requirements better than bringing the existing workload to Azure.
- **Rebuild:** Start over with a new implementation when the cost of the aforementioned strategies outweighs the benefits. Rebuilding makes sense for legacy workloads that need fundamental changes to run well in the cloud. For example, you might rebuild a custom CRM system using Dynamics 365 if the existing codebase is difficult to maintain and doesn't align well with Azure services.
- **Retain:** Keep the workload on-premises when compliance, latency, or technical constraints make migration impractical. For example, you might retain a legacy mainframe system that can't be easily rehosted or refactored and doesn't have a clear migration path to Azure.

Most workload migrations in the Azure Migration Hub use a **rehost or replatform** approach. The goal is a like-for-like migration that minimizes risk: the workload should meet the same KPIs, SLAs, and SLOs on Azure that it met on the source platform. It's best to wait for migration to complete, before you spend time on optimization and modernization.

For more information on migration strategies and when to use each one, see [Select a cloud migration strategy](/azure/cloud-adoption-framework/plan/select-cloud-migration-strategy).

## The migration journey

Every migration follows five phases. Some phases overlap, and you might revisit earlier phases as you learn more, but the sequence helps you track progress.

| Phase | What happens | Outcome |
|-------|-------------|----------------------|
| **Plan** | Assess your current workload, identify dependencies, map source services to Azure equivalents, and define success criteria. | A clear picture of what you're migrating, what changes are required, and what "done" looks like. |
| **Prepare** | Set up your Azure environment: landing zones, networking, identity, and governance. Design the target-state architecture. | Azure foundation ready to receive the workload. You resolve architectural decisions before execution starts. |
| **Execute** | Migrate infrastructure, data, and application components. Perform iterative testing and cutover. | Workload components move to Azure. Upon successful testing of the workload, traffic is redirected to Azure. |
| **Evaluate** | Validate that the migrated workload meets functional, performance, security, and cost requirements against the baseline you set in Phase 1. | Confirmation that the migration succeeded and the workload runs correctly on Azure. |
| **Decommission** | Retire the source environment. Remove resources, cancel subscriptions, and close out the old platform. | Source workload shut down. Azure is now the single platform for this workload. |

## Migration guidance

The section lists the types of  migration guidance that Azure provides. Each guide is designed to help you plan and manage your migration.

### Cloud Adoption Framework

[Cloud Adoption Framework](/azure/cloud-adoption-framework/migrate/plan-migration) covers organization-level planning: how to structure your migration, what steps to take, and what to set up before you move workloads.

If you're new to Azure, start here. The Cloud Adoption Framework walks you through organizational preparation, from building your Azure enrollment and setting up a platform landing zone to creating a high-level migration plan. Get these in place before you start moving workloads.

### Azure Architecture Center

The [Azure Architecture Center](/azure/architecture/browse/) has solution ideas, architectures, design patterns, and architecture guides for building workloads on Azure.

Most migrations involve replatforming: you move both the infrastructure and management layer from your source cloud to Azure. Not all source components have a direct Azure equivalent, so you might need to redesign parts of the architecture. The Architecture Center provides an overview over the [technology choices](/azure/architecture/guide/technology-choices/technology-choices-overview/) available and helps you find the closest match.

### Well-Architected Framework

[Well-Architected Framework](/azure/well-architected/) gives you principles for building reliable, secure, cost-effective, and efficient cloud systems.

In addition to general architecture advise, it also contains service guides for individual Azure services. These guides give you core best practices to help you make architectural decisions for your workload. Use them to evaluate your architecture after migration and find areas to improve.

## Start with your source platform

Each category includes comparison articles. To get started, compare the capabilities of your workload and its services with their closest Azure counterparts. These articles also include example scenarios and service-level migration guides to illustrate the comparisons.

> [!div class="nextstepaction"]
> [Migrate from Amazon Web Services (AWS) to Azure](./migrate-from-aws.yml)

> [!div class="nextstepaction"]
> [Migrate from Google Cloud (GCP) to Azure](./migrate-from-google-cloud.yml)

> [!div class="nextstepaction"]
> [Migrate from on-premises to Azure](./migrate-from-on-premises.yml)


## Tools for migration

These source system agnostic tools help with migration tasks and measuring success against business goals.

| Tool | Purpose |
|------|---------|
| [Azure Migrate and Modernize](/azure/migrate/migrate-services-overview) | Discover and assess migration assets, including infrastructure, applications, and data components. |
| Well-Architected Review assessment of the source platform | Review and measure the business goals of your architecture on the source platform. This assessment, provided by your source cloud provider, helps you establish a baseline for your expectations on Azure. |
| [Azure Well-Architected Review assessment](/assessments/azure-architecture-review/) | Evaluate your architecture decisions to identify regressions from the source baseline and explore optimization opportunities. |

## Need help getting started?

If you're not sure about a migration strategy or need organization-level planning before migrating individual workloads, start here:

- [Plan your migration](/azure/cloud-adoption-framework/migrate/plan-migration). The Cloud Adoption Framework covers migration sequencing, wave planning, dependency mapping, and stakeholder alignment.