---
title: "Tutorial: Connect to ServiceNow in Azure SRE Agent"
description: Configure ServiceNow as your incident platform using basic authentication or OAuth 2.0 for automated incident management.
ms.topic: tutorial
ms.service: azure-sre-agent
ms.date: 03/30/2026
author: craigshoemaker
ms.author: cshoe
ms.ai-usage: ai-assisted
ms.custom: servicenow, itsm, incidents, integration, oauth, oauth2, connector, incident platform
#customer intent: As an SRE or IT operations engineer, I want to connect my ServiceNow instance to Azure SRE Agent so that I can automate incident response and management.
---

# Tutorial: Connect to ServiceNow in Azure SRE Agent

Connect your ServiceNow instance to receive and manage incidents automatically. Choose between **Basic Authentication** (username/password) or **OAuth 2.0** (token-based, recommended for production).

**Time**: 15 minutes

## Prerequisites

- An Azure SRE Agent [created and running](create-and-set-up.md)
- A ServiceNow instance with admin access
- For **OAuth 2.0**: An OAuth application registered in ServiceNow (see [Prepare ServiceNow for OAuth](#prepare-servicenow-for-oauth))
- For **Basic Authentication**: A ServiceNow user with incident management permissions

## Choose your authentication method

| Method | Security | Best for | Setup time |
|--------|----------|----------|------------|
| **OAuth 2.0** | Token-based, automatic refresh, no stored passwords | Production environments, security-conscious teams | ~10 min |
| **Basic Authentication** | Username and password stored | Quick setup, dev/test environments | ~5 min |

> [!NOTE]
> OAuth 2.0 is recommended for production use. Tokens refresh automatically and no passwords are stored in your agent configuration.

## Option A: Connect with OAuth 2.0

### Prepare ServiceNow for OAuth

Before connecting from the portal, register an OAuth application in your ServiceNow instance:

1. In your ServiceNow instance, navigate to **System OAuth** > **Application Registry**.
2. Select **New** > **Create an OAuth API endpoint for external clients**.
3. Fill in:
   - **Name**: A descriptive name (for example, `Azure SRE Agent`)
   - **Redirect URL**: `https://logic-apis-{region}.consent.azure-apim.net/redirect` (where `{region}` is your agent's Azure region, for example, `eastus2`)
   - **Active**: Checked
4. Select **Submit** and note the **Client ID** and **Client Secret**.

> [!WARNING]
> The Client Secret is shown only once in some ServiceNow versions. Copy it before navigating away.

### Configure in the portal

1. Navigate to your agent in the [Azure SRE Agent portal](https://sre.azure.com).
2. In the left navigation, go to **Settings** > **Incident platform**.
3. Select **ServiceNow** from the **Incident platform** dropdown.
4. Set **Authentication Type** to **OAuth 2.0**.
5. A yellow info box appears with a redirect URL. Copy this URL and add it to your ServiceNow OAuth application.
6. Enter your ServiceNow details:
   - **ServiceNow endpoint**: Your instance URL (for example, `https://your-instance.service-now.com`)
   - **OAuth Client ID**: From your ServiceNow OAuth application
   - **OAuth Client Secret**: From your ServiceNow OAuth application
7. Select **Authorize**.

### Authorize the connection

1. The portal creates an Azure API Connection and opens a ServiceNow sign-in popup.
2. Sign in to ServiceNow to authorize the connection.
3. Wait for authorization to complete.
4. When authorization succeeds, the **Authorize** button changes to **Save**.
5. Select **Save** to save the configuration.

### Verify your connection

After authorization, the settings page shows a green status indicator with **"ServiceNow is connected."**

## Option B: Connect with basic authentication

### Create a ServiceNow integration user

1. In ServiceNow, go to **User Administration** > **Users**.
2. Create a new user with:
   - A recognizable username (for example, `sre-agent-integration`)
   - A strong password
   - Roles: `itil` and incident management permissions

### Configure in the portal

1. Navigate to your agent in the [Azure SRE Agent portal](https://sre.azure.com).
2. In the left navigation, go to **Settings** > **Incident platform**.
3. Select **ServiceNow** from the dropdown.
4. Leave **Authentication Type** as **Basic Authentication** (default).
5. Enter your ServiceNow details:
   - **ServiceNow endpoint**: Your instance URL
   - **Username**: The integration user's username
   - **Password**: The integration user's password
6. Select **Save**.

### Verify your connection

The portal validates connectivity automatically. You should see a green status indicator with **"ServiceNow is connected."**

## Set up response plans

Once ServiceNow is connected, [create response plans](incident-response-plans.md) to define how your agent handles incidents.

When you enable **Quickstart response plan**, your agent creates a default plan:

| Platform | Default plan handles | Autonomy level |
|----------|---------------------|----------------|
| **ServiceNow** | High (Priority 2) incidents | Autonomous |

ServiceNow priority values:

| Priority | Value | Label |
|----------|-------|-------|
| Critical | 1 | Critical |
| High | 2 | High |
| Moderate | 3 | Moderate |
| Low | 4 | Low |
| Planning | 5 | Planning |

## What your agent can do with ServiceNow

| Capability | Description |
|-----------|-------------|
| **Read incidents** | Fetch incident details, related records, and discussion history |
| **Post discussion entries** | Add investigation findings and updates to incident work notes |
| **Acknowledge incidents** | Mark incidents as acknowledged when investigation begins |
| **Change priority** | Adjust incident priority based on investigation findings |
| **Resolve incidents** | Close incidents with resolution notes after successful mitigation |

## Troubleshooting

| Issue | Resolution |
|-------|------------|
| OAuth authorization fails | Verify the redirect URL matches exactly and the OAuth application is active |
| Connection shows "Not Connected" | Re-authorize OAuth or verify basic auth credentials are correct |
| "Unable to connect to ServiceNow endpoint" | Verify the URL format (`https://your-instance.service-now.com`, no trailing slash) |
| "Invalid OAuth credentials" | Regenerate the Client Secret in ServiceNow and try again |

## Related content

- [Incident platforms](incident-platforms.md)
- [Incident response](incident-response.md)
- [Incident response plans](incident-response-plans.md)
