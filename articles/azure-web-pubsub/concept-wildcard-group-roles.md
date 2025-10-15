---
title: Use wildcard group role patterns
description: Learn how to grant Azure Web PubSub clients permissions to many groups using wildcard role patterns.
author: kevinguo-ed
ms.author: kevinguo
ms.service: azure-web-pubsub
ms.topic: conceptual
ms.date: 10/14/2025
ms.custom:
---

# Use wildcard group role patterns

Azure Web PubSub now supports wildcard pattern matching in client "group" roles so you can authorize a client for many related groups with a single role string. This reduces token size, simplifies permission management, and improves performance versus enumerating many concrete group roles.

You can continue to use the existing literal roles:

- `webpubsub.sendToGroup.{groupName}`
- `webpubsub.joinLeaveGroup.{groupName}`

But you can now also use the new pattern roles:

- `webpubsub.sendToGroups.{pattern}`
- `webpubsub.joinLeaveGroups.{pattern}`

Where `{pattern}` follows the wildcard syntax below.

## When to use pattern roles

Use pattern roles when:

- A user or device must access a large but bounded dynamic set of groups (for example: all groups for a specific tenant or project)
- You want to keep access tokens small (avoid listing dozens or hundreds of explicit group roles)

Avoid over-broad patterns (like `**`) unless absolutely required; follow the principle of least privilege.

## Pattern syntax

| Symbol | Meaning |
| ------ | ------- |
| `?` | Matches exactly one character except `/` |
| `*` | Matches zero or more characters except `/` |
| `**` | Matches zero or more characters including `/` (crosses path boundaries) |
| `\` | Escape character for `\`, `*`, `?` |

Additional rules:

- `/` acts as a path separator and is never matched by `?` or `*` (only by `**`).
- Use `**` sparingly; prefer narrower patterns (`clientA/*/chat`).
- Up to five total `*` characters (including those forming `**`) are allowed in a single pattern.

### Examples

| Pattern | Matches | Does not match |
| ------- | ------- | -------------- |
| `chat-*` | `chat-1`, `chat-room` | `chat/1`, `xchat-1` |
| `clientA/*` | `clientA/alpha`, `clientA/1` | `clientA/alpha/room1`, `clientB/alpha` |
| `clientA/**` | `clientA/alpha`, `clientA/alpha/room1` | `clientB/anything` |
| `clientA/rooms/?1` | `clientA/rooms/a1`, `clientA/rooms/11` | `clientA/rooms/1`, `clientA/rooms/a/1` |
| `literal\*star` | `literal*star` | `literalXstar` |

### Escaping

Prefix `*`, `?`, or `\` with `\` to match the literal character. Example: `project\*123` matches only `project*123`.

## Using pattern roles in code

Add the pattern role to the `roles` collection when generating a client access token. The client then automatically has the implied permissions for matching groups.

## Code samples

# [JavaScript](#tab/javascript)

```js
const token = await serviceClient.getClientAccessToken({
  roles: [
    // Can send to all groups under clientA/
  'webpubsub.sendToGroups.clientA/**',
    // Can join/leave any direct child group under clientA/public/
  'webpubsub.joinLeaveGroups.clientA/public/*'
  ]
});
```

# [C#](#tab/csharp)

```csharp
var url = service.GetClientAccessUri(roles: new [] {
  "webpubsub.sendToGroups.clientA/**",
  "webpubsub.joinLeaveGroups.clientA/public/*"
});
```

# [Python](#tab/python)

```python
token = service.get_client_access_token(roles=[
  "webpubsub.sendToGroups.clientA/**",
  "webpubsub.joinLeaveGroups.clientA/public/*"
])
```

# [Java](#tab/java)

```java
GetClientAccessTokenOptions opt = new GetClientAccessTokenOptions();
opt.addRole("webpubsub.sendToGroups.clientA/**");
opt.addRole("webpubsub.joinLeaveGroups.clientA/public/*");
WebPubSubClientAccessToken token = service.getClientAccessToken(opt);
```

---

## Security guidance

- Prefer the narrowest pattern that satisfies the scenario.

## Frequently asked questions

**Q: Can I mix literal and pattern roles?**
Yes. A literal role always applies exactly; patterns add broader coverage.


## Next steps

> [!div class="nextstepaction"]
> [Generate client access URL and use roles](howto-generate-client-access-url.md)

> [!div class="nextstepaction"]
> [Authorize access with Microsoft Entra ID](concept-azure-ad-authorization.md)
