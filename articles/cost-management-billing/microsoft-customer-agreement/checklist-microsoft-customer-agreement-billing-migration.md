---
title: Checklist Microsoft Customer Agreement Billing Migration
description: This guide helps customers who have signed a Microsoft Customer Agreement prepare their existing subscriptions for a billing migration.
author: Nicholak-MS
ms.service: cost-management-billing
ms.subservice: microsoft-customer-agreement
ms.topic: conceptual
ms.date: 12/31/2025
ms.author: nicholak
ms.reviewer: nicholak
ms.custom:
---

# MCA Transition Billing Migration Checklist

## Overview

Before migrating from an Enterprise Agreement (EA), Microsoft Customer Agreement (MCA), or Pay-As-You-Go (PAYG) subscriptions to a Microsoft Customer Agreement (MCA), review this checklist and follow the required steps to ensure a smooth transition. This checklist helps you:

- Validate readiness and dependencies
- Minimize post-migration issues
- Align stakeholders on required actions

## Validate Contract and Roles

Confirm access to both the source platform and the destination MCA as a Billing Account Owner.

- EA → MCA: Ensure EA Admin and MCA Billing Account Owner roles are assigned.
- PAYG → MCA: Ensure a Global Admin for the PAYG subscription and MCA Billing Account Owner role in the destination account.
- MCA → MCA: Confirm Billing Account Owner roles exist in both source and destination MCA billing accounts.

## Download Historical Data

- Export historical cost and usage data before migration. Historical data does not transfer to MCA.
- Save invoices and custom reports for compliance.
- [View and download Azure usage and charges - Microsoft Cost Management | Microsoft Learn](../cost-management-billing/understand/download-azure-daily-usage)

## Review Billing Hierarchy Changes

- Understand the MCA structure: Billing Account → Billing Profile → Invoice Section → Subscription
- Map existing departments or subscriptions to MCA invoice sections.

:::image type="content" source="./media/onboard-microsoft-customer-agreement/mca-structure.png" alt-text="Diagram showing the structure of a Microsoft Customer Agreement." lightbox="./media/onboard-microsoft-customer-agreement/mca-structure.png" :::

## Identify Reservations and Savings Plans

### Reservations

Self-service reservation transfers: Supported when there is no currency change.

- Currency change scenario:
  - If there's a currency change during or after enrollment transfer, monthly-paid reservations are cancelled for the source enrollment.
  - Cancellation occurs at the time of the next monthly payment for each individual reservation.
  - This cancellation is intentional and only affects monthly reservation purchases.

### Azure Savings Plan

USD currency savings plans: Transfer automatically during migration.

- Non-USD currency savings plans:
  - Savings Plans from the source enrollment will not transfer.
  - They will be cancelled in the source enrollment and automatically repurchased in the destination enrollment.

- Important details for repurchased Savings Plans:
  - Each new Savings Plan will be billed monthly, regardless of the original billing frequency.
  - Each new Savings Plan will be priced as the USD equivalent of the original plan (e.g., €5/hour → $5.85/hour at €1:$1.17).
  - Each new Savings Plan will have a 1-year term, even if the original was 3 years.
  - If the original plan was 1-year, savings benefits remain the same.
  - If moving from 3-year to 1-year, expect reduced savings benefits due to discount differences.
    - To maintain previous savings levels, purchase an additional 1-year Savings Plan.
    - Hourly commitment recommendations for additional plans may take up to 2 days to appear in the Azure portal.
  - Customers with 3-year plans who want to retain discounts should immediately contact Azure Support to purchase new 3-year plans in the destination enrollment.

- For more details, please review: [Azure product transfer hub - Microsoft Cost Management | Microsoft Learn](../cost-management-billing/manage/subscription-transfer#product-transfer-support)

## Cost Management & Reporting

- Recreate the following under MCA:
  - Budgets
  - Alerts
  - Exports
  - Custom/shared views
  - Cost Allocation rules
- Update Power BI connectors:
  - Use Billing Profile ID instead of EA enrollment number.
  - [Connect to Microsoft Cost Management data in Power BI Desktop](https://docs.microsoft.com/power-bi/connect-data/desktop-connect-azure-cost-management)

## API & Automation Updates

Replace legacy APIs with MCA APIs and updated billing properties. APIs & Automation need to be recreated in MCA.

- Update automation scripts for:
  - Subscription vending
  - Auto-purchases
  - Third-party cost tools (e.g., CloudHealth)

## Technical Dependencies

- Validate Terraform or ARM templates for subscription creation.
- Check compatibility of dashboards (e.g., Emissions Impact Dashboard) and update IDs to MCA billing scope.

## Invoice & Payment Setup

- MCA remit-to information differs from EA or PAYG.
- Notify accounts payable team.
- Create separate records for EA and MCA invoices.
- Expect a final invoice from the source platform and new monthly MCA invoices.
- For bank details verification letters, contact your Microsoft Account team.

## Tax & Compliance

- Tax exemption certificates: If your account has a tax exemption certificate, create an Azure support request to associate it with your MCA account.
- Billing profile alignment and currency usage: The billing profile's sold-to and bill-to country/region must correspond to the MCA market country/region.

## Support Plan

- Support plans do not transfer to MCA.
- Cancel existing plans per contract terms.
- Purchase new MCA support plan if required.
- Migration may affect Unified Support subscriptions—contact your Microsoft representative.

## Next Steps

- [Set up billing for Microsoft Customer Agreement.](../cost-management-billing/manage/mca-setup-account#before-you-start-the-setup-we-recommend-you-do-the-following-actions)
- [Onboard to the Microsoft Customer Agreement (MCA).](../microsoft-customer-agreement/onboard-microsoft-customer-agreement#migrate-from-an-ea-to-an-mca)
