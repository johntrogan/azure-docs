---
title: Manage an Informatica resource through the Azure portal
description: Learn how to manage the Informatica Intelligent Data Management Cloud Azure Native Service through the Azure portal. 
ms.topic: how-to
ms.date: 12/11/2025
#customer intent: As a developer, I want to use the Azure portal to manage single sign-on for my Informatica IDMC service and how to remove a deployment.
---

# Manage your Informatica organization through the Azure portal

In this article about the Informatica Intelligent Data Management Cloud (IDMC) Azure Native Service, you learn how to manage single sign-on for your organization. You also learn how to delete an Informatica deployment.

## Single sign-on

Single sign-on (SSO) is already enabled when you created your Informatica Organization. To access Organization through SSO, follow these steps:

1. Navigate to the **Overview** for your instance of the Informatica organization. Select the single sign-on URL, or select **IDMC Account Login**.

   :::image type="content" source="media/informatica-manage/informatica-sso-overview.png" alt-text="Screenshot showing the Single Sign-on URL in the Overview pane of the Informatica resource." lightbox="media/informatica-manage/informatica-sso-overview.png":::

1. The first time you access this URL, depending on your Azure tenant settings, you might see a request to grant permissions and user consent.

   > [!NOTE]
   > If you also see an administrator consent screen, check your [tenant consent settings](/azure/active-directory/manage-apps/configure-user-consent).

1. Choose a Microsoft Entra account for single sign-on. After you provide consent, the setup procedure redirects you to the Informatica portal.

## Delete an Informatica deployment

After you delete the Informatica resource, all billing stops for that resource through Azure Marketplace. If you're done using your resource and want to delete that resource, follow these steps:

1. From the service menu, select the Informatica deployment that you want to delete.

1. On the working pane of the **Overview**, select **Delete**.

    :::image type="content" source="media/informatica-manage/informatica-delete-overview.png" alt-text="Screenshot showing how to delete an Informatica resource." lightbox="media/informatica-manage/informatica-delete-overview.png":::

1. Confirm that you want to delete the Informatica resource by entering the name of the resource.

    :::image type="content" source="media/informatica-manage/informatica-confirm-delete.png" alt-text="Screenshot showing the final confirmation of delete for an Informatica resource." lightbox="media/informatica-manage/informatica-confirm-delete.png":::

1. Select the reason why you want to delete the resource.

1. Select **Delete**.

## Get support

Contact [Informatica](https://support.informatica.com/) for customer support. 

You can also request support in the Azure portal from the resource overview:

- Select **Support + Troubleshooting** > **New support request** from the service menu. Then choose the link to the [Informatica support website](https://support.informatica.com/).
