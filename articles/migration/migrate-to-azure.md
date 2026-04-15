---
title: Migrate Workloads to Azure
description: Learn about migration resources that might help you transition workloads from AWS, GCP and on-premises to Azure.
author: reginahack
ms.author: rhackenberg
ms.date: 03/24/2025
ms.topic: concept-article
ms.service: azure
ms.custom: migration-hub
---

# Migrate workloads to Azure

You're migrating workloads to Azure. This article helps you find the right starting point based on where your workloads run today.

The Azure Migration Hub provides prescriptive guides for migrating individual workloads from other clouds or on-premises infrastructure to Azure. Each guide follows five phases: Plan, Prepare, Execute, Evaluate, and Decommission.

> [!IMPORTANT]
> This content covers single-workload migrations. It doesn't cover full datacenter migrations, region relocations, or hybrid workloads that run concurrently on multiple clouds.
The migration journey

## Target audience

This content is for workload teams responsible for planning and executing migrations:

- **Workload architects** who redesign architecture aspects and validate the overall design to meet business requirements on Azure.
- **Workload team members** who need to understand how their responsibilities change during and after migration.

## The migration journey

Every migration follows five phases. Some phases overlap, and you might revisit earlier phases as you learn more, but the sequence gives you a reliable framework to track progress.

| Phase | What happens | What you should expect |
|-------|-------------|----------------------|
| **1. Plan** | Assess your current workload, identify dependencies, map source services to Azure equivalents, and define success criteria. | You build a clear picture of what you're migrating, what changes are required, and what "done" looks like. |
| **2. Prepare** | Set up your Azure environment: landing zones, networking, identity, and governance. Design the target-state architecture. | Your Azure foundation is ready to receive the workload. You resolve architectural decisions before execution starts. |
| **3. Execute** | Migrate infrastructure, data, and application components. Perform iterative testing and cutover. | Workload components move to Azure. You validate each component before moving to the next. |
| **4. Evaluate** | Validate that the migrated workload meets functional, performance, security, and cost requirements against the baseline you set in Phase 1. | You confirm the migration succeeded and the workload is running correctly on Azure. |
| **5. Decommission** | Retire the source environment. Remove resources, cancel subscriptions, and close out the old platform. | The source workload is shut down. Azure is your single platform for this workload. |

## Where are your workloads today?

### Migrate from Amazon Web Services (AWS)

You're running on AWS and moving workloads to Azure.

> [!div class="nextstepaction"]
> [Migrate a workload from AWS to Azure](./migrate-workload-from-aws-introduction)

This prescriptive guide walks you through the full migration life cycle for a single workload, from planning through decommissioning your AWS resources.

To compare AWS services with their Azure equivalents, see [AWS to Azure service comparisons](./migrate-from-aws.yml).

### Migrate from Google Cloud

You're running on Google Cloud and moving workloads to Azure.

> [!div class="nextstepaction"]
> [Migrate from Google Cloud to Azure](./migrate-from-google-cloud.yml)

Compare Google Cloud services with their Azure equivalents and review migration scenarios.

### Migrate from on-premises

You're running in your own datacenter and moving workloads to Azure.

> [!div class="nextstepaction"]
> [Migrate from on-premises to Azure](./migrate-from-on-premises.yml)

Review migration scenarios for infrastructure and databases. A prescriptive on-premises-to-Azure workload migration guide is in development.
## Tools

Use these tools to assist with migration tasks or measure success against business goals.

| Tool                                                                                | Purpose                                                                                                                                                           |
| ----------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Azure Migrate and Modernize](/azure/migrate/migrate-services-overview)             | Discover and assess migration assets, including infrastructure, applications, and data components.                                                                |
| Well-Architected Review assessment of the source platform, if available             | Review and measure the business goals of your architecture on the source platform. This assessment helps you establish a baseline for your expectations on Azure. |
| [Azure Well-Architected Review assessment](/assessments/azure-architecture-review/) | Evaluate your architecture decisions to identify regressions from the source baseline and explore optimization opportunities.                                     |

## Microsoft frameworks that guide the migration journey

Migrations are often complex and require a significant amount of planning and preparation. Microsoft developed three frameworks that help guide aspects of the migration journey:

- [Cloud Adoption Framework (CAF)](https://review.learn.microsoft.com/en-us/azure/cloud-adoption-framework/migrate/plan-migration)
- [Azure Architecture Center (AAC)](https://review.learn.microsoft.com/en-us/azure/architecture/guide/migration/migration-start-here)
- [Well-Architected Framework (WAF)](https://review.learn.microsoft.com/en-us/azure/well-architected/)

If you're new to Azure, start with **CAF**. It provides organization-level guidance, helps you understand the migration process, identify the necessary steps, and implement best practices to ensure a successful migration. It shows how to prepare your organization for migration. Migrate workloads only after your organization is committed to Azure and you established your approach for adopting Azure. Before migrating workloads, understand the fundamental concepts on Azure, have an active Azure enrollment as well as a platform landing zone, and a high-level migration plan.

Once you have the plan and foundation in place, the **Azure Architecture Center (AAC)** provides guidance on how to design and implement your workloads. It includes reference architectures, best practices, and design patterns for building solutions on Azure.

Migrations to Azure typically involve _replatforming the workload_, which includes transitioning both the infrastructure and management layer from the source cloud provider to Azure. To prepare for the migration process, find the best match for your source components on Azure. Keep in mind that not all components have direct equivalents. You need to redesign the architecture or revisit code to maintain functionality and accomplish your business objectives. The Azure Architecture Center (AAC) offers comparisons of the typical workload components and platform services.

The **Well-Architected Framework** is a set of guiding principles that help you design and operate reliable, secure, efficient, and cost-effective systems in the cloud. It provides a structured approach to evaluate your architecture and identify areas for improvement.
## Not sure where to start?

If you aren't sure about a migration strategy or need organization-level planning before migrating individual workloads, start here:

- [Plan your migration](/azure/cloud-adoption-framework/migrate/plan-migration). The Cloud Adoption Framework covers migration sequencing, wave planning, dependency mapping, and stakeholder alignment.
- [Select a migration strategy](/azure/cloud-adoption-framework/plan/select-cloud-migration-strategy). Understand the trade-offs between rehost, replatform, refactor, rebuild, replace, and retain.
- [Migration readiness assessment](/assessments/Strategic-Migration-Assessment/). Score your readiness across 10 dimensions before you begin.
- [Azure fundamental concepts](/azure/cloud-adoption-framework/ready/considerations/fundamental-concepts). Learn about terms used in Azure and how the concepts relate to one another.
- [Cloud Adoption Framework Migrate methodology](/training/modules/cloud-adoption-framework-migrate/). Complete this training module to develop your organization's migration plan.