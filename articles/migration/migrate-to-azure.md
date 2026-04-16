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

The Azure Migration Hub provides prescriptive, opinionated guidance to help workload teams plan and implement migrations to Azure. It covers migrations from on‑premises environments and cloud platforms such as Amazon Web Services (AWS) and Google Cloud Platform (GCP).

The intended outcome is a completed migration to Azure, followed by decommissioning the workload on the source platform.

> [!IMPORTANT]
> This content covers single-workload migrations. It doesn't cover full datacenter migrations, region relocations, or hybrid workloads that run concurrently on multiple clouds.

Migration to Azure isn't a single product decision. It touches networking, identity, databases, compute, storage, and custom glue your team built over the years. The guidance for all of this lives across different articles and guides. 

Each follows five phases: Plan, Prepare, Execute, Evaluate, and Decommission.

This article exists so you don't have to figure out which of those sources applies to you. It does one thing: asks where your workloads are today, then sends you to the right guide. It also covers general migration terminology and strategies.

## Who should read it

This content is for workload teams responsible for planning and executing migrations:

- **Workload architects** who redesign architecture aspects and validate the overall design to meet business requirements on Azure. Architects address gaps by considering the workload's specific characteristics and business constraints.
- **Workload team members** who need to understand how their responsibilities change during and after migration. For example, database administrators who manage scripts and perform daily backups on Amazon Relational Database Service must adapt to performing the same tasks on Azure SQL Database.

Read this article if you're a workload architect or engineer who was told "we're moving to Azure" and you need to know where to start. You might be coming from AWS, Google Cloud, or an on-premises datacenter. You might not know yet whether you should rehost, replatform, or refactor. 

If you're a manager looking for a business case or cost justification, this article isn't for you. 

## What you get from reading it

In about two minutes, you learn:

- Which prescriptive guide matches your source platform (AWS, GCP, or on-premises)
- What the five migration phases are and what to expect in each one.
- Which migration strategy fits your workload (rehost, replatform, refactor, or rebuild).
- Where to go if you need organizational planning before you touch any workloads.
- What tools are available to help with discovery, assessment, and validation.

This article doesn't provide a step-by-step migration walkthrough. For that information, see the linked guides. This article helps you find the right guide and saves you from reading three frameworks to figure out which one you actually need.

## Why it matters

Every week a workload team spends searching for the right starting point is a week they aren't migrating. The teams that move fastest are the ones that skip the research phase of "which Microsoft docs do I read first" and go straight to the guide that matches their scenario. This page helps you find that guide.

## Migration strategies - 6Rs

Choose a strategy for each workload based on its complexity, your timeline, and how much you want to change during the move.

- **Rehost (lift and shift):** Move the workload as-is to Azure infrastructure without modifying code. This approach is fast and low risk. It works well for straightforward workloads where the priority is getting off the source platform quickly.
- **Replatform (lift, tinker, and shift):** Make minimal changes to take advantage of Azure platform services. For example, migrate a SQL Server database to Azure SQL Managed Instance. You get operational benefits without a full rewrite.
- **Refactor:** Restructure existing code to improve performance, scalability, or maintainability without changing the workload's external behavior. Refactoring takes more effort up front but improves the workload's long-term viability on Azure.
- **Rebuild:** Start over with a new implementation when the cost of replatforming or refactoring outweighs the benefits. Rebuilding makes sense for legacy workloads that need fundamental changes to run well in the cloud.
- **Replace:** Adopt a third-party cloud service instead of migrating your existing implementation. Consider this option when a SaaS or PaaS solution meets your requirements better than bringing the existing workload to Azure.
- **Retain:** Keep the workload on-premises when compliance, latency, or technical constraints make migration impractical.

Most workload migrations covered in this hub use a **rehost or replatform** approach. The goal is a like-for-like migration that minimizes risk: the workload should meet the same KPIs, SLAs, and SLOs on Azure that it met on the source platform. Save optimization and modernization for after the migration is complete.

For a deeper look at migration strategies and when to use each one, see [Select a cloud migration strategy](/azure/cloud-adoption-framework/plan/select-cloud-migration-strategy).

## The migration journey

Every migration follows five phases. Some phases overlap, and you might revisit earlier phases as you learn more, but the sequence helps you track progress.

| Phase | What happens | What you should expect |
|-------|-------------|----------------------|
| **1. Plan** | Assess your current workload, identify dependencies, map source services to Azure equivalents, and define success criteria. | A clear picture of what you're migrating, what changes are required, and what "done" looks like. |
| **2. Prepare** | Set up your Azure environment: landing zones, networking, identity, and governance. Design the target-state architecture. | Azure foundation ready to receive the workload. You resolve architectural decisions before execution starts. |
| **3. Execute** | Migrate infrastructure, data, and application components. Perform iterative testing and cutover. | Workload components move to Azure. You validate each component before moving to the next. |
| **4. Evaluate** | Validate that the migrated workload meets functional, performance, security, and cost requirements against the baseline you set in Phase 1. | Confirmation that the migration succeeded and the workload runs correctly on Azure. |
| **5. Decommission** | Retire the source environment. Remove resources, cancel subscriptions, and close out the old platform. | Source workload shut down. Azure is now the single platform for this workload. |

## Frameworks

Microsoft offers three frameworks that cover different parts of the migration process.

### Cloud Adoption Framework

[Cloud Adoption Framework](/azure/cloud-adoption-framework/migrate/plan-migration) covers organization-level planning: how to structure your migration, what steps to take, and what to set up before you move workloads.

If you're new to Azure, start here. CAF walks you through organizational preparation, from building your Azure enrollment and setting up a platform landing zone to creating a high-level migration plan. Get these in place before you start moving workloads.

### Azure Architecture Center

The [Azure Architecture Center](/azure/architecture/browse/) has solution ideas, architectures, design patterns, and architecture guides for building workloads on Azure.

Most migrations involve replatforming: you move both the infrastructure and management layer from your source cloud to Azure. Not all source components have a direct Azure equivalent, so you might need to redesign parts of the architecture. The Architecture Center provides an overview over the [technology choices](/azure/architecture/guide/technology-choices/technology-choices-overview/) available and helps you find the closest match.

### Well-Architected Framework

[Well-Architected Framework](/azure/well-architected/) gives you principles for building reliable, secure, efficient, and cost-effective cloud systems. Use it to evaluate your architecture after migration and find areas to improve.

## Choose your source platform

### Migrate from Amazon Web Services (AWS)

This prescriptive guide walks you through the full migration life cycle for a single workload, from planning through decommissioning your AWS resources.

> [!div class="nextstepaction"]
> [Migrate a workload from AWS to Azure](/azure/migration/migrate-workload-from-aws-introduction)

To compare AWS services with their Azure equivalents, see [AWS to Azure service comparisons](./migrate-from-aws.yml).

### Migrate from Google Cloud

Compare Google Cloud services with their Azure equivalents and review migration scenarios.

> [!div class="nextstepaction"]
> [Migrate from Google Cloud to Azure](./migrate-from-google-cloud.yml)

### Migrate from on-premises

Review migration scenarios for infrastructure and databases. A prescriptive on-premises-to-Azure workload migration guide is in development.

> [!div class="nextstepaction"]
> [Migrate from on-premises to Azure](./migrate-from-on-premises.yml)


## Tools for migration

These tools help with migration tasks and measuring success against business goals.

| Tool | Purpose |
|------|---------|
| [Azure Migrate and Modernize](/azure/migrate/migrate-services-overview) | Discover and assess migration assets, including infrastructure, applications, and data components. |
| Well-Architected Review assessment of the source platform | Review and measure the business goals of your architecture on the source platform. This assessment, provided by your source cloud provider, helps you establish a baseline for your expectations on Azure. |
| [Azure Well-Architected Review assessment](/assessments/azure-architecture-review/) | Evaluate your architecture decisions to identify regressions from the source baseline and explore optimization opportunities. |

## Need help getting started?

If you're not sure about a migration strategy or need organization-level planning before migrating individual workloads, start here:

- [Plan your migration](/azure/cloud-adoption-framework/migrate/plan-migration). The Cloud Adoption Framework covers migration sequencing, wave planning, dependency mapping, and stakeholder alignment.
- [Select a migration strategy](/azure/cloud-adoption-framework/plan/select-cloud-migration-strategy). Understand the trade-offs between rehost, replatform, refactor, rebuild, replace, and retain.
- [Migration readiness assessment](/assessments/Strategic-Migration-Assessment/). Score your readiness across 10 dimensions before you begin.
- [Azure fundamental concepts](/azure/cloud-adoption-framework/ready/considerations/fundamental-concepts). Learn about terms used in Azure and how the concepts relate to one another.
- [Cloud Adoption Framework Migrate methodology](/training/modules/cloud-adoption-framework-migrate/). Complete this training module to develop your organization's migration plan.