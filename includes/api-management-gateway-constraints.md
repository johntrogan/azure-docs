---
title: API Management gateway runtime limits
description: Include file
services: api-management
author: dlepow

ms.service: azure-api-management
ms.topic: include
ms.date: 04/08/2026
ms.author: danlep
ms.custom: Include file
---

<!-- Constraints - API Management gateways  -->


| Runtime limit | Consumption | Classic tiers | v2 tiers |
| ---------| ----------- | ----------- | ----------- |
| Concurrent back-end connections<sup>1</sup> per HTTP authority | Unlimited | 2,048<sup>2</sup> per unit | 2,048 |
| Cached response size | 2 MiB | 2 MiB | 2 MiB |
| Policy document size  | 16 KiB | 512 KiB | 512 KiB |
| Request payload size | 1 GiB | Unlimited | 1 GiB |
| Buffered payload size | 2 MiB | 500 MiB | 2 MiB |
| Request/response payload size in diagnostic logs | 8,192 bytes | 8,192 bytes | 8,192 bytes |
| Request URL size<sup>3</sup> | 16,384 bytes | Unlimited | 16,384 bytes |
| Length of URL path segment | 1,024 characters | 1,024 characters | 1,024 characters |
| Length of named value | 4,096 characters | 4,096 characters | 4,096 characters |
| Size of request or response body in [validate-content policy](/azure/api-management/validate-content-policy) | 100 KiB | 100 KiB | 100 KiB |
| Size of API schema used by [validation policy](/azure/api-management/validation-policies) | 4 MB | 4 MB | 4 MB |
| Active WebSocket connections per unit<sup>4</sup> | N/A | 5,000 | 5,000 |

<sup>1</sup> Connections are pooled and reused unless explicitly closed by the backend.<br/>
<sup>2</sup> Limit is 1,024 in the Developer tier.<br/>
<sup>3</sup> Includes an up to 2048-bytes long query string.<br/>
<sup>4</sup> Up to a maximum of 60,000 connections per service instance.

