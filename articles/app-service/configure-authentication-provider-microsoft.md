---
title: Configure Azure App Service and Azure Functions for Microsoft Account Sign In
description: Learn how to configure Microsoft account authentication as an identity provider for your App Service or Azure Functions app.
ms.assetid: ffbc6064-edf6-474d-971c-695598fd08bf
ms.topic: how-to
ms.date: 03/19/2026
ms.custom: fasttrack-edit, AppServiceIdentity
author: cephalin
ms.author: cephalin
ms.service: azure-app-service
# customer intent: As a developer, I want to configure Microsoft account authentication so that I can use it as an identity provider for App Service and Azure Functions apps. 
  
---

# Configure your App Service or Azure Functions app to use Microsoft account sign-in

[!INCLUDE [app-service-mobile-selector-authentication](../../includes/app-service-mobile-selector-authentication.md)]

This article describes how to configure Azure App Service or Azure Functions to use Microsoft Entra ID to support personal Microsoft account sign-ins.

> [!IMPORTANT]
> Although the Microsoft account provider is still supported, we recommend that you use the [Microsoft identity platform provider (Microsoft Entra ID)](./configure-authentication-provider-aad.md) for your apps. The Microsoft identity platform provides support for both organizational accounts and personal Microsoft accounts.

## <a name="register-microsoft-account"> </a>Register your app with Microsoft account

To register your app with Microsoft Account: 

1. Go to [**App registrations**](https://portal.azure.com/#blade/Microsoft_AAD_RegisteredApps/ApplicationsListBlade) in the Azure portal. As needed, sign in with your Microsoft account.
1. Select **New registration**, and then enter an application name.
1. Under **Supported account types**, select **Any Entra ID Tenant + Personal Microsoft Accounts**.
1. In **Redirect URIs**, select **Web**, and then enter `https://<app-domain-name>/.auth/login/aad/callback`. Replace `<app-domain-name>` with the domain name of your app.  For example, `https://contoso.azurewebsites.net/.auth/login/aad/callback`. Be sure to use the HTTPS scheme in the URL.

1. Select **Register**.
1. Copy the **Application (Client) ID**. You'll need it later.
1. In the left pane, select **Management** > **Certificates & secrets**. Select **New client secret**. Enter a description, select the expriration date, and then select **Add**.
1. Copy the value that appears on the **Certificates & secrets** page. After you leave the page, it won't be displayed again.

    > [!IMPORTANT]
    > The client secret value (password) is an important security credential. Don't share the password with anyone or distribute it within a client application.

## <a name="secrets"> </a>Add Microsoft account information to your App Service application

To add Microsoft account information to your App Service application: 

1. Go to your application in the [Azure portal].
1. Select **Settings** > **Authentication**. 
1. Select **Add identity provider**.
1. Under **Identity provider**, select **Microsoft**.  
1. Select **External configuration**.
1. Select **Provide the details of an existing app registration**.
1. Paste in the **Application (client) ID** and **Client secret** that you obtained earlier. Use https://login.microsoftonline.com/9188040d-6c67-4c5b-b112-36a304b66dad/v2.0 for the **Issuer URL** field.


   App Service provides authentication but doesn't restrict authorized access to your site content and APIs. You must authorize users in your app code.

1. (Optional) To restrict access to Microsoft account users, set **Unauthenticated requests** to **HTTP 302 Found redirect: recommended for websites**. When you configure this option, your app requires all requests to be authenticated. It also redirects all unauthenticated requests to use Microsoft Entra ID for authentication. Note that because you have configured your **Issuer URL** to use the Microsoft account tenant, only personal accounts will successfully authenticate.

   > [!CAUTION]
   > Restricting access in this way applies to all calls to your app. This configuration might not be desirable for apps that have a publicly available home page, as in many single-page applications. For such applications, **Allow anonymous requests (no action)** might be preferred so that the app manually starts authentication itself. For more information, see [Authentication flow](overview-authentication-authorization.md#authentication-flow).

1. Select **Add**.

You can now use Microsoft Account for authentication in your app.

## <a name="related-content"> </a>Related content

[!INCLUDE [app-service-mobile-related-content-get-started-users](../../includes/app-service-mobile-related-content-get-started-users.md)]

<!-- URLs. -->

[My Applications]: https://go.microsoft.com/fwlink/p/?LinkId=262039
[Azure portal]: https://portal.azure.com/
