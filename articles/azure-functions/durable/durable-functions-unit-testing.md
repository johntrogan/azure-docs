---
title: Azure Durable Functions unit testing
description: Learn how to unit test Durable Functions.
ms.topic: conceptual
ms.date: 02/13/2026
---

# Durable Functions unit testing (C# in-process)

Unit testing is an important part of modern software development practices. Unit tests verify business logic behavior and protect from introducing unnoticed breaking changes in the future. Durable Functions can easily grow in complexity so introducing unit tests helps avoid breaking changes. The following sections explain how to unit test the three function types - Orchestration client, orchestrator, and activity functions.

> [!NOTE]
> This article provides guidance for unit testing for Durable Functions apps written in C# for the .NET in-process worker and targeting Durable Functions 2.x. For more information about the differences between versions, see the [Durable Functions versions](durable-functions-versions.md) article.

## Prerequisites

The examples in this article require knowledge of the following concepts and frameworks:

* Unit testing

* Durable Functions

* [xUnit](https://github.com/xunit/xunit) - Testing framework

* [moq](https://github.com/moq/moq4) - Mocking framework

## Base classes for mocking

Mocking is supported via the following interface:

* [IDurableOrchestrationClient](/dotnet/api/microsoft.azure.webjobs.extensions.durabletask.idurableorchestrationclient), [IDurableEntityClient](/dotnet/api/microsoft.azure.webjobs.extensions.durabletask.idurableentityclient), and [IDurableClient](/dotnet/api/microsoft.azure.webjobs.extensions.durabletask.idurableclient)

* [IDurableOrchestrationContext](/dotnet/api/microsoft.azure.webjobs.extensions.durabletask.idurableorchestrationcontext)

* [IDurableActivityContext](/dotnet/api/microsoft.azure.webjobs.extensions.durabletask.idurableactivitycontext)

* [IDurableEntityContext](/dotnet/api/microsoft.azure.webjobs.extensions.durabletask.idurableentitycontext)

These interfaces can be used with the various trigger and bindings supported by Durable Functions. While it is executing your Azure Functions, the functions runtime runs your function code with a concrete implementation of these interfaces. For unit testing, you can pass in a mocked version of these interfaces to test your business logic.

## Unit testing trigger functions

In this section, the unit test validates the logic of the following HTTP trigger function for starting new orchestrations.

```csharp
using System.Net.Http;
using System.Threading.Tasks;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.DurableTask;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.Extensions.Logging;

namespace VSSample
{
    public static class HttpStart
    {
        [FunctionName("HttpStart")]
        public static async Task<HttpResponseMessage> Run(
            [HttpTrigger(AuthorizationLevel.Function, "post", Route = "orchestrators/{functionName}")] HttpRequestMessage req,
            [DurableClient] IDurableClient starter,
            string functionName,
            ILogger log)
        {
            object eventData = await req.Content.ReadAsAsync<object>();
            string instanceId = await starter.StartNewAsync(functionName, eventData);

            log.LogInformation($"Started orchestration with ID = '{instanceId}'.");

            return starter.CreateCheckStatusResponse(req, instanceId);
        }
    }
}
```

The unit test task verifies the value of the `Retry-After` header provided in the response payload. So the unit test mocks some of `IDurableClient` methods to ensure predictable behavior.

First, we use a mocking framework ([moq](https://github.com/moq/moq4) in this case) to mock `IDurableClient`:

```csharp
// Mock IDurableClient
var durableClientMock = new Mock<IDurableClient>();
```

> [!NOTE]
> While you can mock interfaces by directly implementing the interface as a class, mocking frameworks simplify the process in various ways. For instance, if a new method is added to the interface across minor releases, moq doesn't require any code changes unlike concrete implementations.

Then `StartNewAsync` method is mocked to return a well-known instance ID.

```csharp
// Mock StartNewAsync method
durableClientMock.
    Setup(x => x.StartNewAsync(functionName, It.IsAny<object>())).
    ReturnsAsync(instanceId);
```

Next, the test needs to handle `CreateCheckStatusResponse`. Since `CreateCheckStatusResponse` is an extension method, it can't be mocked directly with Moq. Instead, mock the underlying `CreateHttpManagementPayload` method, which is an instance method on `IDurableClient`:

```csharp
// CreateCheckStatusResponse is an extension method and cannot be mocked directly.
// Mock CreateHttpManagementPayload, which is the underlying instance method.
durableClientMock
    .Setup(x => x.CreateHttpManagementPayload(instanceId))
    .Returns(new HttpManagementPayload
    {
        Id = instanceId,
        StatusQueryGetUri = $"http://localhost:7071/runtime/webhooks/durabletask/instances/{instanceId}",
        SendEventPostUri = $"http://localhost:7071/runtime/webhooks/durabletask/instances/{instanceId}/raiseEvent/{{eventName}}",
        TerminatePostUri = $"http://localhost:7071/runtime/webhooks/durabletask/instances/{instanceId}/terminate",
        PurgeHistoryDeleteUri = $"http://localhost:7071/runtime/webhooks/durabletask/instances/{instanceId}"
    });
```

> [!NOTE]
> `CreateCheckStatusResponse` is an extension method that internally calls `CreateHttpManagementPayload`. Extension methods are static and cannot be mocked using standard mocking frameworks like Moq. By mocking `CreateHttpManagementPayload`, you control the data used by the extension method.

`ILogger` is also mocked:

```csharp
// Mock ILogger
var loggerMock = new Mock<ILogger>();
```

Now the `Run` method is called from the unit test:

```csharp
// Call Orchestration trigger function
var result = await HttpStart.Run(
    new HttpRequestMessage()
    {
        Content = new StringContent("{}", Encoding.UTF8, "application/json"),
        RequestUri = new Uri("http://localhost:7071/orchestrators/E1_HelloSequence"),
    },
    durableClientMock.Object,
    functionName,
    loggerMock.Object);
```

 The last step is to compare the output with the expected value:

```csharp
// Validate the response status code
Assert.Equal(HttpStatusCode.OK, result.StatusCode);
```

After you combine all these steps, the unit test has the following code:

```csharp
using System;
using System.Net;
using System.Net.Http;
using System.Text;
using System.Threading.Tasks;
using Microsoft.Azure.WebJobs.Extensions.DurableTask;
using Microsoft.Extensions.Logging;
using Moq;
using Xunit;

namespace VSSample.Tests
{
    public class HttpStartTests
    {
        [Fact]
        public async Task HttpStart_returns_management_payload()
        {
            // Arrange
            string instanceId = "7E467BDB-213F-407A-B86A-1954053D3C24";
            string functionName = "E1_HelloSequence";

            var durableClientMock = new Mock<IDurableClient>();

            durableClientMock
                .Setup(x => x.StartNewAsync(functionName, It.IsAny<object>()))
                .ReturnsAsync(instanceId);

            // CreateCheckStatusResponse is an extension method and cannot be mocked directly.
            // Mock CreateHttpManagementPayload, which is the underlying instance method.
            durableClientMock
                .Setup(x => x.CreateHttpManagementPayload(instanceId))
                .Returns(new HttpManagementPayload
                {
                    Id = instanceId,
                    StatusQueryGetUri = $"http://localhost:7071/runtime/webhooks/durabletask/instances/{instanceId}",
                    SendEventPostUri = $"http://localhost:7071/runtime/webhooks/durabletask/instances/{instanceId}/raiseEvent/{{eventName}}",
                    TerminatePostUri = $"http://localhost:7071/runtime/webhooks/durabletask/instances/{instanceId}/terminate",
                    PurgeHistoryDeleteUri = $"http://localhost:7071/runtime/webhooks/durabletask/instances/{instanceId}"
                });

            var loggerMock = new Mock<ILogger>();

            // Act
            var result = await HttpStart.Run(
                new HttpRequestMessage
                {
                    Content = new StringContent("{}", Encoding.UTF8, "application/json"),
                    RequestUri = new Uri("http://localhost:7071/orchestrators/E1_HelloSequence"),
                },
                durableClientMock.Object,
                functionName,
                loggerMock.Object);

            // Assert
            Assert.Equal(HttpStatusCode.OK, result.StatusCode);
        }
    }
}
```

## Unit testing orchestrator functions

Orchestrator functions are even more interesting for unit testing since they usually have a lot more business logic.

In this section, the unit tests validate the output of the `E1_HelloSequence` Orchestrator function:

```csharp
using System.Collections.Generic;
using System.Threading.Tasks;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.DurableTask;

namespace VSSample
{
    public static class HelloSequence
    {
        [FunctionName("E1_HelloSequence")]
        public static async Task<List<string>> Run(
            [OrchestrationTrigger] IDurableOrchestrationContext context)
        {
            var outputs = new List<string>();

            outputs.Add(await context.CallActivityAsync<string>("E1_SayHello", "Tokyo"));
            outputs.Add(await context.CallActivityAsync<string>("E1_SayHello", "Seattle"));
            outputs.Add(await context.CallActivityAsync<string>("E1_SayHello", "London"));

            return outputs;
        }

        [FunctionName("E1_SayHello")]
        public static string SayHello([ActivityTrigger] IDurableActivityContext context)
        {
            string name = context.GetInput<string>();
            return $"Hello {name}!";
        }
    }
}
```

The unit test code starts with creating a mock:

```csharp
var durableOrchestrationContextMock = new Mock<IDurableOrchestrationContext>();
```

Then the activity method calls are mocked:

```csharp
durableOrchestrationContextMock.Setup(x => x.CallActivityAsync<string>("E1_SayHello", "Tokyo")).ReturnsAsync("Hello Tokyo!");
durableOrchestrationContextMock.Setup(x => x.CallActivityAsync<string>("E1_SayHello", "Seattle")).ReturnsAsync("Hello Seattle!");
durableOrchestrationContextMock.Setup(x => x.CallActivityAsync<string>("E1_SayHello", "London")).ReturnsAsync("Hello London!");
```

Next, the unit test calls the `HelloSequence.Run` method:

```csharp
var result = await HelloSequence.Run(durableOrchestrationContextMock.Object);
```

And finally the output is validated:

```csharp
Assert.Equal(3, result.Count);
Assert.Equal("Hello Tokyo!", result[0]);
Assert.Equal("Hello Seattle!", result[1]);
Assert.Equal("Hello London!", result[2]);
```

After you combine the previous steps, the unit test has the following code:

```csharp
using System.Collections.Generic;
using System.Threading.Tasks;
using Microsoft.Azure.WebJobs.Extensions.DurableTask;
using Moq;
using Xunit;

namespace VSSample.Tests
{
    public class HelloSequenceOrchestratorTests
    {
        [Fact]
        public async Task Run_returns_multiple_greetings()
        {
            var durableOrchestrationContextMock = new Mock<IDurableOrchestrationContext>();
            durableOrchestrationContextMock.Setup(x => x.CallActivityAsync<string>("E1_SayHello", "Tokyo")).ReturnsAsync("Hello Tokyo!");
            durableOrchestrationContextMock.Setup(x => x.CallActivityAsync<string>("E1_SayHello", "Seattle")).ReturnsAsync("Hello Seattle!");
            durableOrchestrationContextMock.Setup(x => x.CallActivityAsync<string>("E1_SayHello", "London")).ReturnsAsync("Hello London!");

            var result = await HelloSequence.Run(durableOrchestrationContextMock.Object);

            Assert.Equal(3, result.Count);
            Assert.Equal("Hello Tokyo!", result[0]);
            Assert.Equal("Hello Seattle!", result[1]);
            Assert.Equal("Hello London!", result[2]);
        }
    }
}
```

## Unit testing activity functions

Activity functions are unit tested in the same way as nondurable functions.

In this section the unit test validates the behavior of the `E1_SayHello` Activity function:

```csharp
[FunctionName("E1_SayHello")]
public static string SayHello([ActivityTrigger] IDurableActivityContext context)
{
    string name = context.GetInput<string>();
    return $"Hello {name}!";
}
```

And the unit tests verify the format of the output. These unit tests either use the parameter types directly or mock `IDurableActivityContext` class:

```csharp
using Microsoft.Azure.WebJobs.Extensions.DurableTask;
using Moq;
using Xunit;

namespace VSSample.Tests
{
    public class HelloSequenceActivityTests
    {
        [Fact]
        public void SayHello_returns_greeting()
        {
            var durableActivityContextMock = new Mock<IDurableActivityContext>();
            durableActivityContextMock.Setup(x => x.GetInput<string>()).Returns("World");

            var result = HelloSequence.SayHello(durableActivityContextMock.Object);

            Assert.Equal("Hello World!", result);
        }
    }
}
```

## Next steps

> [!div class="nextstepaction"]
> [Learn more about xUnit](https://xunit.net/docs/getting-started/netcore/cmdline)
>
> [Learn more about moq](https://github.com/Moq/moq4/wiki/Quickstart)
