---
title: Add Web Page Knowledge Sources in Azure SRE Agent
description: Learn how to add publicly accessible web pages as knowledge sources in Azure SRE Agent.
ms.topic: concept-article
ms.service: azure-sre-agent
ms.date: 03/30/2026
author: dm-chelupati
ms.author: dchelupati
ms.ai-usage: ai-assisted
ms.custom: knowledge, web-page, URL, knowledge-base, external-resources
---

# Add web page knowledge sources in Azure SRE Agent

Add any publicly accessible web page by URL—your agent fetches and indexes the content automatically so it can reference external documentation, status pages, and wiki articles during investigations.

> [!TIP]
> - Add any publicly accessible web page by URL—your agent fetches and indexes the content automatically
> - Reference external documentation, status pages, runbook sites, and wiki articles during investigations
> - One-step setup: provide a URL, name, and optional description

## The problem: external docs stay external

Your team's knowledge isn't all in one place. Runbooks live on wiki sites. Vendor documentation is on external portals. Architecture diagrams and status pages are scattered across different URLs. When your agent investigates an issue, it can't reference these external resources—unless someone manually copies the content and uploads it as a file.

That manual process is tedious and creates stale copies. The original page gets updated, but the uploaded file doesn't.

## How web page knowledge works

When you add a web page as a knowledge source, the agent fetches the page content and stores it for reference:

1. You provide a URL, a name, and an optional description.
2. The agent fetches the page content via an anonymous HTTP request.
3. The page content is stored and indexed.
4. Your agent can reference this content during conversations and investigations.

The fetch happens at the time you add the URL. Content is stored as a point-in-time snapshot of the page.

The agent fetches pages anonymously—without authentication credentials. Pages that require login, VPN access, or corporate SSO can't be indexed. If you need to add content from protected pages, copy the content and [upload it as a file](upload-knowledge-document.md) instead.

## What gets indexed

| Aspect | Behavior |
|--------|----------|
| **Content fetched** | Full page content from the single URL provided |
| **Link following** | No—only the specified URL is fetched, not linked pages |
| **Authentication** | Anonymous—no credentials sent with the request |
| **Supported protocols** | HTTP and HTTPS |
| **Fetch timeout** | 30 seconds |
| **Refresh** | Manual—delete and re-add the URL to get updated content |

## When to use web page knowledge

Web page knowledge sources work best for:

- **Public documentation**—vendor docs, API references, cloud service guides
- **Status pages**—service health dashboards, incident history pages
- **Wiki articles**—publicly accessible knowledge base articles
- **Architecture overviews**—publicly hosted architecture diagrams and design docs
- **Runbook sites**—external runbook repositories accessible without authentication

For internal or protected content that requires authentication, use [file uploads](upload-knowledge-document.md) instead.

## Limitations

| Limitation | Details |
|------------|---------|
| **No authentication** | Can't access pages behind login, VPN, or SSO |
| **Single page only** | Doesn't crawl or follow links to other pages |
| **Point-in-time snapshot** | Content isn't automatically refreshed when the source page changes |
| **30-second timeout** | Pages that take longer than 30 seconds to load fail |
| **URL format** | Must be an absolute HTTP or HTTPS URL |

## Related content

- [Connect knowledge](connect-knowledge.md)
- [Upload knowledge documents](upload-knowledge-document.md)
- [File attachments](file-attachments.md)

## Next step

> [!div class="nextstepaction"]
> [Add a web page knowledge source](tutorial-upload-knowledge-document.md)
