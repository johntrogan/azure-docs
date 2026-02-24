---
author: mattchenderson
ms.service: azure-functions
ms.topic: include
ms.date: 07/24/2024
ms.author: mahender
---

## Connections

The `connection` property is a reference to environment configuration which specifies how the app should connect to Service Bus. It may specify:

- The name of an application setting containing a [connection string](#connection-string)
- The name of a shared prefix for multiple application settings, together defining an [identity-based connection](#identity-based-connections).

If the configured value is both an exact match for a single setting and a prefix match for other settings, the exact match is used.

### Connection string

Obtain this connection string by following the steps shown at [Get the management credentials](../articles/service-bus-messaging/service-bus-quickstart-portal.md#get-the-connection-string).  The connection string must be for a Service Bus namespace, not limited to a specific queue or topic.

This connection string should be stored in an application setting with a name matching the value specified by the `connection` property of the binding configuration.

If the app setting name begins with "AzureWebJobs", you can specify only the remainder of the name. For example, if you set `connection` to "MyServiceBus", the Functions runtime looks for an app setting that is named "AzureWebJobsMyServiceBus." If you leave `connection` empty, the Functions runtime uses the default Service Bus connection string in the app setting that is named "AzureWebJobsServiceBus".

### Identity-based connections

If you are using [version 5.x or higher of the extension](../articles/azure-functions/functions-bindings-service-bus.md#service-bus-extension-5x-and-higher), instead of using a connection string with a secret, you can have the app use a [Microsoft Entra identity](../articles/active-directory/fundamentals/active-directory-whatis.md). To do this, you would define settings under a common prefix which maps to the `connection` property in the trigger and binding configuration.

In this mode, the extension requires the following properties:

| Property                  | Environment variable template                       | Description                                | Example value |
|---------------------------|-----------------------------------------------------|--------------------------------------------|---------|
| Fully Qualified Namespace | `<CONNECTION_NAME_PREFIX>__fullyQualifiedNamespace` | The fully qualified Service Bus namespace. | <service_bus_namespace>.servicebus.windows.net  |

Additional properties may be set to customize the connection. See [Common properties for identity-based connections](../articles/azure-functions/functions-reference.md#common-properties-for-identity-based-connections).

#### User-assigned managed identity

To use a user-assigned managed identity, add the `credential` and `clientId` properties in addition to the `fullyQualifiedNamespace`:

| Property                  | Environment variable template                       | Description                                | Example value |
|---------------------------|-----------------------------------------------------|--------------------------------------------|---------|
| Fully Qualified Namespace | `<CONNECTION_NAME_PREFIX>__fullyQualifiedNamespace` | The fully qualified Service Bus namespace. | `myservicebus.servicebus.windows.net`  |
| Credential | `<CONNECTION_NAME_PREFIX>__credential` | Must be set to `managedidentity`. | `managedidentity` |
| Client ID | `<CONNECTION_NAME_PREFIX>__clientId` | The client ID of the user-assigned managed identity. | `00000000-0000-0000-0000-000000000000` |

For example, if your binding configuration specifies `connection = "ServiceBusConnection"`, you would configure the following application settings:

```json
{
    "ServiceBusConnection__fullyQualifiedNamespace": "myservicebus.servicebus.windows.net",
    "ServiceBusConnection__credential": "managedidentity",
    "ServiceBusConnection__clientId": "00000000-0000-0000-0000-000000000000"
}
```

> [!TIP]
> User-assigned managed identities are recommended for production scenarios where you need fine-grained control over identity permissions across multiple resources.
> [!NOTE]
> When using [Azure App Configuration](../articles/azure-app-configuration/quickstart-azure-functions-csharp.md) or [Key Vault](../articles/key-vault/general/overview.md) to provide settings for Managed Identity connections, setting names should use a valid key separator such as `:` or `/` in place of the `__` to ensure names are resolved correctly.
>
> For example: `ServiceBusConnection:fullyQualifiedNamespace`

[!INCLUDE [functions-identity-based-connections-configuration](./functions-identity-based-connections-configuration.md)]

[!INCLUDE [functions-service-bus-permissions](./functions-service-bus-permissions.md)]
