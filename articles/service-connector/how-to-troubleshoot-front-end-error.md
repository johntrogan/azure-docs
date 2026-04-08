---
title: Service Connector troubleshooting
description: See error messages and suggested actions for troubleshooting issues with Service Connector.
author: maud-lv
ms.author: malev
ms.service: service-connector
ms.topic: troubleshooting
ms.date: 04/08/2026
ms.custom: kr2b-contr-experiment, devx-track-azurecli
#customer intent: As a Service Connector user, I need to understand the Service Connector error messages and suggested actions so I can use them to troubleshoot Service Connector.
---

# Service Connector troubleshooting

This article lists Service Connector error messages you might encounter in the Azure portal or Azure CLI, and suggests actions you can take to troubleshoot them.

## Azure portal

The following errors and troubleshooting actions can occur in the Azure portal.

### Unknown resource type

- Check whether Service Connector supports the source and target resource service types. 
- Check whether Service Connector supports the specified source-target connection combination.
- Check whether the target resource exists. 
- Check whether the target resource ID is correct. 

### Unsupported resource

- Check whether the selected authentication type supports the specified source-target connection combination. 

## Azure CLI

The following errors and troubleshooting actions can occur in Azure CLI.

### InvalidArgumentValue errors

| Error message  | Suggested action  |
|--------------------|-----------------------------|
| `The source resource ID is invalid: {SourceId}` | - Check whether Service Connector supports the source resource ID.<br> - Check whether the source resouce ID is correct. |
| `Target resource ID is invalid: {TargetId}`     | - Check whether Service Connector supports the target service type.<br> - Check whether the target resouce ID is correct. |
| `Connection ID is invalid: {ConnectionId}`      | Check whether the connection ID is correct.  |

#### RequiredArgumentMissing errors

| Error message  | Suggested action  |
|--------------------|-----------------------------|
| `{Argument} shouldn't be blank`    | Provide an argument value for interactive input.     |
| `Required keys missing for parameter {Parameter}. All possible keys are: {Keys}` | Provide a value for the authentication information parameter, usually in the form of `--param key1=val1 key2=val2`. |
| `Required argument is missing, please provide the arguments: {Arguments}`          | Provide the required argument.  |

#### Validation errors

| Error message  | Suggested action  |
|--------------------|-----------------------------|
| `Only one auth info is needed` | Provide one and only one authentication information parameter. Check whether there are none, or more than one. |
| `Auth info argument should be provided when updating the connection: {ConnectionName}`  | Provide the authentication information when you update a secret connection type. The Azure Resource Manager API can't access a user's secret. |
| `Either client type or auth info should be specified to update`  | Provide either client type or authentication information when you update a connection.  |
| `Usage error: {} [KEY=VALUE ...]` | Check the available keys and provide values for the authentication information parameter, usually in the form of `--param key1=val1 key2=val2`. |
| `Unsupported Key {Key} is provided for parameter {Parameter}. All possible keys are: {Keys}` | Check the available keys and provide values for the authentication information parameter, usually in the form of `--param key1=val1 key2=val2`. |
| `Provision failed, please create the target resource manually and then create the connection. Error details: {ErrorTrace}` | Retry. If necessary, create the target resource manually and then create the connection.  |

## Related content

- [Service Connector concepts](./concept-service-connector-internals.md)
- [Service Connector known limitations](./known-limitations.md)
