---
title: Checklist Microsoft Customer Agreement Billing Migration
description: This guide helps customers who sign a Microsoft Customer Agreement prepare their existing subscriptions for a billing migration.
author: Nicholak-MS
ms.service: cost-management-billing
ms.subservice: microsoft-customer-agreement
ms.topic: article
ms.date: 1/12/2026
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
- Top customer actions with MCA [Learn More](https://www.microsoft.com/licensing/news/top_customer_actions_after_accepting_microsoft_customer_agreement?rtc=1)

## Validate Contract and Roles

Confirm access to both the source platform and the destination MCA as a Billing Account Owner.

- EA → MCA: Ensure EA Admin and MCA Billing Account Owner roles are assigned.
- PAYG → MCA: Ensure a Global Admin for the PAYG subscription and MCA Billing Account Owner role in the destination account.
- MCA → MCA: Confirm Billing Account Owner roles exist in both source and destination MCA billing accounts.

## Download Historical Data

- Export historical cost and usage data before migration. Historical data doesn't transfer to MCA.
- Save invoices and custom reports for compliance.
- [View and download Azure usage and charges - Microsoft Cost Management | Microsoft Learn](https://learn.microsoft.com/azure/cost-management-billing/understand/download-azure-daily-usage)

## Review Billing Hierarchy Changes

- Understand the MCA structure: Billing Account → Billing Profile → Invoice Section → Subscription
- Map existing departments or subscriptions to MCA invoice sections.

:::image type="content" source="./media/onboard-microsoft-customer-agreement/mca-structure.jpg" alt-text="Diagram showing the structure of a Microsoft Customer Agreement." lightbox="./media/onboard-microsoft-customer-agreement/mca-structure.jpg" :::

## Identify Reservations and Savings Plans

### Reservations

Self-service reservation transfers: Supported when there's no currency change.

- Currency change scenario:
  - If there's a currency change during or after enrollment transfer, monthly paid reservations are canceled for the source enrollment.
  - Cancellation occurs at the time of the next monthly payment for each individual reservation.
  - This cancellation is intentional and only affects monthly reservation purchases.

### Azure Savings Plan

USD currency savings plans: Transfer automatically during migration.

- Non-USD currency savings plans:
  - Savings Plans from the source enrollment won't transfer.
  - They'll be canceled in the source enrollment and automatically repurchased in the destination enrollment.

- Important details for repurchased Savings Plans:
  - Each new Savings Plan is billed monthly, regardless of the original billing frequency.
  - Each new Savings Plan is priced as the USD equivalent of the original plan (for example, €5/hour → $5.85/hour at €1:$1.17). [Learn More](https://learn.microsoft.com/azure/cost-management-billing/manage/mca-request-billing-ownership#prerequisites)
  - Each new Savings Plan has a one year term, even if the original was three years.
  - If the original plan was one year, savings benefits remain the same.
  - If moving from three year to one year, expect reduced savings benefits due to discount differences.
    - To maintain previous savings discount levels, work with your account management team to purchase another one year Savings Plan with adjusted discounts. 
    - Hourly commitment recommendations for new plans may take up to two days to appear in the Azure portal.
  - Customers with three year plans who want to retain discounts should immediately contact Azure Support to purchase new three year plans in the destination enrollment.

- For more details, please review: [Azure product transfer hub - Microsoft Cost Management | Microsoft Learn](https://learn.microsoft.com/azure/cost-management-billing/manage/subscription-transfer#product-transfer-support)

## Cost Management & Reporting

- Recreate the following aspects under MCA:
  - Budgets
  - Alerts
  - Exports
  - Custom/shared views
  - Cost Allocation rules
- Update Power BI connect:
  - Use Billing Profile ID instead of EA enrollment number.
  - [Connect to Microsoft Cost Management data in Power BI Desktop](https://docs.microsoft.com/power-bi/connect-data/desktop-connect-azure-cost-management)

## API & Automation Updates

Replace legacy APIs with MCA APIs and updated billing properties. APIs & Automation need to be recreated in MCA.

- Update automation scripts for:
  - Update any programming code to replace EA API calls with MCA API calls. [Learn More](https://learn.microsoft.com/azure/cost-management-billing/costs/migrate-cost-management-api)
  - Subscription vending
  - Automatic purchases
  - Third-party cost tools (for example, CloudHealth)
 
>!Note!
> EA and MCA API schemas differ. [Learn More](https://learn.microsoft.com/azure/cost-management-billing/costs/migrate-cost-management-api#apis-to-get-cost-and-usage)

## Technical Dependencies

- EA to MCA migration is an evolutionary experience involving contract & technical changes
- Validate Terraform or ARM templates for subscription creation.
- Check compatibility of dashboards (for example, Emissions Impact Dashboard) and update references to MCA billing scope.

## Invoice Setup

- Changes in billing constructs
  - Getting started with MCA billing [Learn More](https://learn.microsoft.com/azure/cost-management-billing/understand/mca-overview)
  - Organizing your invoice based on your needs [Learn More](https://learn.microsoft.com/azure/cost-management-billing/manage/mca-section-invoice#structure-your-account-with-billing-profiles-and-invoice-sections)

## Payment Setup

- MCA remit-to information differs from EA or PAYG.[Learn More[ (https://learn.microsoft.com/azure/cost-management-billing/manage/mca-section-invoice#structure-your-account-with-billing-profiles-and-invoice-sections)
- Notify accounts payable team.
- Create separate records for EA and MCA invoices. 
- Expect a final invoice from the source platform and new monthly MCA invoices. [Learn More](https://learn.microsoft.com/azure/cost-management-billing/manage/mca-section-invoice#structure-your-account-with-billing-profiles-and-invoice-sections)
- For bank details verification letters, contact your Microsoft Account team.

## Tax & Compliance

- Tax exemption certificates: If your account has a tax exemption certificate, create an Azure support request to associate it with your MCA account.
- Billing profile alignment and currency usage: The billing profile's sold-to and bill-to country/region must correspond to the MCA market country/region.

## Support Plan

- Support plans don't transfer to MCA. You will need to re-purcahse a Support Plan on your MCA. 
- Cancel existing plans per contract terms, otherwise you will continue to be billed until the end of your contract terms. 
- Migration may affect Unified Support subscriptions, contact your Microsoft Account Team. 

## Next Steps

- [Set up billing for Microsoft Customer Agreement.](https://learn.microsoft.com/azure/cost-management-billing/manage/mca-setup-account#before-you-start-the-setup-we-recommend-you-do-the-following-actions)
- [Onboard to the Microsoft Customer Agreement (MCA).](https://learn.microsoft.com/azure/cost-management-billing/microsoft-customer-agreement/onboard-microsoft-customer-agreement#migrate-from-an-ea-to-an-mca)
