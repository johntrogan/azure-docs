---
title: Migrate Workloads to Azure
description: Learn about migration resources that might help you transition workloads from AWS and Google Cloud to Azure.
author: reginahack
ms.author: rhackenberg
ms.date: 03/24/2025
ms.topic: concept-article
ms.service: azure
ms.custom: migration-hub
---

# Migrate workloads to Azure from other cloud platforms

The Azure Migration Hub provides content to help workload teams plan and implement their workload migration. It covers migrations from on-premises and cloud platforms, like Amazon Web Services (AWS) and Google Cloud Platform (GCP), to Microsoft Azure. The expected outcome is that, after you complete your migration to Azure, you decommission the workload on the source platform.

> [!IMPORTANT]
>
> Some migration scenarios are out of scope for this content. It doesn't cover full datacenter migrations or region relocations. It also doesn't address hybrid workloads that concurrently run on multiple clouds.

## Target audience

The content in the Azure Migration Hub applies to the following workload roles and functions at the team level.

- **Workload architects** who might redesign various architecture aspects and validate the overall architecture to ensure that it continues to meet business requirements. Architects address gaps by considering the workload's specific characteristics and business constraints.

- **Workload team members** who need to understand how their responsibilities change during the migration process and after migration. For example, database administrators who manage scripts and perform daily backups on Amazon Relational Database Service must adapt to performing the same tasks on Azure SQL Database.

## Microsoft frameworks that guide the migration journey

Migrations are often complex and require a significant amount of planning and preparation. Microsoft developed three frameworks that help guide aspects of the migration journey:

- [Cloud Adoption Framework (CAF)](/azure/cloud-adoption-framework/migrate/plan-migration)
- [Azure Architecture Center (AAC)](/azure/architecture/guide/migration/migration-start-here)
- [Well-Architected Framework (WAF)](/azure/well-architected/)

If you're new to Azure, start with **CAF**. It provides organization-level guidance, helps you understand the migration process, identify the necessary steps, and implement best practices to ensure a successful migration. It shows how to prepare your organization for migration. Migrate workloads only after your organization is committed to Azure and you established your approach for adopting Azure. Before migrating workloads, understand the fundamental concepts on Azure, have an active Azure enrollment as well as a platform landing zone, and a high-level migration plan.

Once you have the plan and foundation in place, the **Azure Architecture Center (AAC)** provides guidance on how to design and implement your workloads. It includes reference architectures, best practices, and design patterns for building solutions on Azure.

Migrations to Azure typically involve _replatforming the workload_, which includes transitioning both the infrastructure and management layer from the source cloud provider to Azure. To prepare for the migration process, find the best match for your source components on Azure. Keep in mind that not all components have direct equivalents. You need to redesign the architecture or revisit code to maintain functionality and accomplish your business objectives. The Azure Architecture Center (AAC) offers comparisons of the typical workload components and platform services.

The **Well-Architected Framework** is a set of guiding principles that help you design and operate reliable, secure, efficient, and cost-effective systems in the cloud. It provides a structured approach to evaluate your architecture and identify areas for improvement.

## Content layout

The Migration Hub content is categorized by the source platform where your workload currently runs. Each category includes comparison articles. To get started, compare the capabilities of your workload and its services with their closest Azure counterparts. These articles also include example scenarios and service-level migration guides to illustrate the comparisons.

Start your learning journey based on your source platform:

> [!div class="nextstepaction"]
> [Migrate a workload from AWS](./migrate-from-aws.yml)

> [!div class="nextstepaction"]
> [Migrate a workload from GCP](./migrate-from-google-cloud.yml)

> [!div class="nextstepaction"]
> [Migrate a workload from on-premises](./migrate-from-on-premises.yml)

This collection also includes articles that apply to all platforms. All sections include such platform-agnostic articles for convenience.

## Tools

In addition to content, you can explore specialized tools to assist with migration tasks or measure the success of your migration against business goals.

|Tool|Purpose|
|---|---|
|[Azure Migrate and Modernize](/azure/migrate/migrate-services-overview)| Discover and assess migration assets, primarily infrastructure, applications, and data components. |
|Well-Architected Review assessment of the source platform, if available| Review and measure the business goals of your architecture on the source platform. This assessment helps you establish a baseline for your expectations on Azure.|
|[Azure Well-Architected Review assessment](/assessments/azure-architecture-review/)| Evaluate your architecture decisions to identify any regressions from the source baseline. Also explore optimization opportunities to get the best on Azure.|