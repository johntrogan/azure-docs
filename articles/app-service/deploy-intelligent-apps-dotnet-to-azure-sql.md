---
title: 'Deploy a .NET Blazor App Connected to Azure SQL and Azure OpenAI on Azure App Service'
description: Follow a tutorial to connect your Azure SQL database with vectorized embeddings to your Azure App Service app that uses Azure OpenAI.
author: jeffwmartinez
ms.author: jefmarti
ms.date: 03/16/2026
ms.update-cycle: 180-days
ms.topic: tutorial
ms.custom:
  - devx-track-dotnet
  - linux-related-content
  - build-2025
ms.collection: ce-skilling-ai-copilot
ms.service: azure-app-service

# Customer intent: As a developer, I want to connect my Azure SQL database with vectorized embeddings to an Azure App Service .NET Blazor app that uses Azure OpenAI, so my app can use vectorized embeddings for search and querying.
---

# Tutorial: Deploy a .NET Blazor app connected to Azure SQL and Azure OpenAI on Azure App Service

You can use your own SQL data to ground the context of your intelligent app. In this tutorial, you create a retrieval augmented generation (RAG) .NET Blazor application by setting up a hybrid vector search against your Azure SQL database. [Azure SQL vector support](https://devblogs.microsoft.com/azure-sql/announcing-eap-native-vector-support-in-azure-sql-database/) provides new [vector functions](/sql/t-sql/functions/vector-functions-transact-sql) that help manage vector data.

## Prerequisites

- An Azure SQL database with data you can vectorize
- An [Azure OpenAI](/azure/ai-services/openai/quickstart?pivots=programming-language-csharp&tabs=command-line%2Ckeyless%2Ctypescript-keyless%2Cpython#set-up) resource
- A .NET 8 or 9 Blazor web app deployed on Azure App Service

>[!NOTE]
>This example builds on the [Chatbot with Azure App Service and Azure OpenAI tutorial](tutorial-ai-openai-chatbot-dotnet.md) for the web app and Azure OpenAI resources.

## 1. Set up the Blazor web app

Create a basic chat box to interact with.

1. Make sure you have the `Microsoft.SemanticKernel` and `Microsoft.Data.SqlClient` packages installed in your development environment.
1. In your web app project tree, expand *Components* > *Pages*, and create a new file in *Pages* named *OpenAI.razor*.
1. Add the following chat box code to the *OpenAI.razor* file, and save the file.

   ```csharp
   @page "/openai"
   @rendermode InteractiveServer
   @inject Microsoft.Extensions.Configuration.IConfiguration _config
   
   <PageTitle>OpenAI</PageTitle>
   
   <h3>OpenAI input query: </h3>
   <input class="col-sm-4" @bind="userMessage" />
   <button class="btn btn-primary" @onclick="SemanticKernelClient">Send Request</button>
   
   <br />
   <br />
   
   <h4>Server response:</h4> <p>@serverResponse</p>
   
   @code {
   
       @using Microsoft.SemanticKernel;
       @using Microsoft.SemanticKernel.ChatCompletion;
       
       }
   ```

## 2. Set up the Azure OpenAI client

After adding the chat interface, set up the Azure OpenAI client using Semantic Kernel. The following code creates the client that connects to your Azure OpenAI resource.

This code needs to use the key and endpoint information for your Azure OpenAI resource. See [Use Key Vault references as app settings in Azure App Service and Azure Functions](app-service-key-vault-references.md) to manage and handle your Azure OpenAI secrets.

>[!NOTE]
>Although not required, it's recommended to use managed identity to secure your client without having to manage API keys. Follow the [.NET Blazor app with OpenAI tutorial](tutorial-ai-openai-chatbot-dotnet.md) for instructions on setting up your Azure OpenAI client to use managed identity.

Add the following code to the *OpenAI.razor* file:

```csharp
@inject Microsoft.Extensions.Configuration.IConfiguration _config

@code {

    @using Microsoft.SemanticKernel;
    @using Microsoft.SemanticKernel.ChatCompletion;

    private string? userMessage;
    private string? serverResponse;

    private async Task SemanticKernelClient()
    {
    
        // App settings
        string deploymentName = _config["DEPLOYMENT_NAME"];
        string endpoint = _config["ENDPOINT"];
        string apiKey = _config["API_KEY"];
        string modelId = _config["MODEL_ID"];

        var builder = Kernel.CreateBuilder();

        // Chat completion service
        builder.Services.AddAzureOpenAIChatCompletion(
            deploymentName: deploymentName,
            endpoint: endpoint,
            apiKey: apiKey,
            modelId: modelId
        );

        var kernel = builder.Build();

        // Create prompt template
        var chat = kernel.CreateFunctionFromPrompt(
            @"{{$history}}
            User: {{$request}}
            Assistant: ");

        ChatHistory chatHistory = new("""You are a helpful assistant that answers questions""");

        var chatResult = kernel.InvokeStreamingAsync<StreamingChatMessageContent>(
                chat,
                new()
                    {
                        { "request", userMessage },
                        { "history", string.Join("\n", chatHistory.Select(x => x.Role + ": " + x.Content)) }
                    }
            );

        string message = "";
        await foreach (var chunk in chatResult)
        {
            message += chunk;
        }

        // Add messages to chat history
        chatHistory.AddUserMessage(userMessage!);
        chatHistory.AddAssistantMessage(message);

        serverResponse = message;
```

You now have a working chat application connected to OpenAI. Next, set up your Azure SQL database to work with your chat application.

## 3. Deploy Azure OpenAI models

To prepare your Azure SQL database for hybrid vector search, you use an embedding model to generate embeddings to use for searching. After you have the appropriate embeddings in your database, you can perform a hybrid vector search on the database that uses the generated embeddings in addition to your initial language model.

This example uses the `text-embedding-ada-002` model to generate embeddings and `gpt-4o-mini` for the language model. Before continuing, use the Microsoft Foundry portal to deploy the two models in your OpenAI resource. For more information, see [Deploy Microsoft Foundry Models in the Foundry portal](/azure/foundry/foundry-models/how-to/deploy-foundry-models).

## 4. Vectorize your Azure SQL database

To perform a hybrid vector search on your Azure SQL database, you must have the appropriate embeddings in your database. Vectorize your database before continuing.

There are many ways to vectorize your database. One option is to use the [Azure SQL `DB` vectorizer](https://github.com/Azure-Samples/azure-sql-db-vectorizer).

## 5. Create a stored procedure that generates embeddings

You can use [Azure SQL vector support](https://devblogs.microsoft.com/azure-sql/announcing-eap-native-vector-support-in-azure-sql-database/) to create a stored procedure that uses a vector data type to store generated embeddings for search queries. The stored procedure invokes an external REST API endpoint to get the embeddings.

The following SQL query creates the stored procedure. Replace the `<resourcename>` placeholder in the `@url` parameter with your Azure OpenAI resource name, and replace the `<openAIkey>` placeholder with the API key from your text embedding model. The model name is part of the `@url`, which populates with the search query.

You can use Visual Studio Code to connect to your database and run the query. For more information, see the Visual Studio Code [MSSQL extension documentation](/sql/tools/visual-studio-code-extensions/mssql/mssql-extension-visual-studio-code).

```sql
CREATE PROCEDURE [dbo].[GET_EMBEDDINGS]
(
    @model VARCHAR(MAX),
    @text NVARCHAR(MAX),
    @embedding VECTOR(1536) OUTPUT
)
AS
BEGIN
    DECLARE @retval INT, @response NVARCHAR(MAX);
    DECLARE @url VARCHAR(MAX);
    DECLARE @payload NVARCHAR(MAX) = JSON_OBJECT('input': @text);

    -- Set the @url variable with proper concatenation before the EXEC statement
    SET @url = 'https://<resourcename>.openai.azure.com/openai/deployments/' + @model + '/embeddings?api-version=2023-03-15-preview';

    EXEC dbo.sp_invoke_external_rest_endpoint 
        @url = @url,
        @method = 'POST',   
        @payload = @payload,   
        @headers = '{"Content-Type":"application/json", "api-key":"<openAIkey>"}', 
        @response = @response OUTPUT;

    -- Use JSON_QUERY to extract the embedding array directly
    DECLARE @jsonArray NVARCHAR(MAX) = JSON_QUERY(@response, '$.result.data[0].embedding');

    
    SET @embedding = CAST(@jsonArray as VECTOR(1536));
END
GO
```

After you create the stored procedure, you can see it under *Stored Procedures* in the *Programmability* folder of your Azure SQL database. You can run a test [similarity search](https://devblogs.microsoft.com/azure-sql/announcing-eap-native-vector-support-in-azure-sql-database/#similarity-search-in-azure-sql-db) in your SQL query editor by using your text embedding model name. This test uses the stored procedure to generate embeddings, calculate vector distance using a vector distance function, and return results based on the text query.

## 6. Connect and search your database

Now that your database is set up to create embeddings, you can connect to the database in your application and set up the hybrid vector search query. Add the following code to your *OpenAI.razor* file, making sure the connection string uses your deployed Azure SQL database connection string. The code uses a SQL parameter that securely passes through the user input from the chat app to the query.

```csharp
// Database connection string
var connectionString = _config["AZURE_SQL_CONNSTRING"];

try
{
    await using var connection = new SqlConnection(connectionString);
    Console.WriteLine("\nQuery results:");

    await connection.OpenAsync();

    // Hybrid search query
    var sql =
        @"DECLARE @e VECTOR(1536);
        EXEC dbo.GET_EMBEDDINGS @model = 'text-embedding-ada-002', @text = '@userMessage', @embedding = @e OUTPUT;

             -- Comprehensive query with multiple filters.
        SELECT TOP(5)
            f.Score,
            f.Summary,
            f.Text,
            VECTOR_DISTANCE('cosine', @e, VectorBinary) AS Distance,
            CASE
                WHEN LEN(f.Text) > 100 THEN 'Detailed Review'
                ELSE 'Short Review'
            END AS ReviewLength,
            CASE
                WHEN f.Score >= 4 THEN 'High Score'
                WHEN f.Score BETWEEN 2 AND 3 THEN 'Medium Score'
                ELSE 'Low Score'
            END AS ScoreCategory
        FROM finefoodembeddings10k$ f
        WHERE
            f.UserId NOT LIKE 'Anonymous%' -- User-based filter to exclude anonymous users
            AND f.Score >= 4 -- Score threshold filter
            AND LEN(f.Text) > 50 -- Text length filter for detailed reviews
            AND (f.Text LIKE '%juice%') -- Inclusion of specific words
        ORDER BY
            Distance,  -- Order by distance
            f.Score DESC, -- Secondary order by review score
            ReviewLength DESC; -- Tertiary order by review length
    ";

    // Set SQL Parameter to pass in user message
    SqlParameter param = new SqlParameter();
    param.ParameterName = "@userMessage";
    param.Value = userMessage;
    
    await using var command = new SqlCommand(sql, connection);

    // add parameter to SqlCommand
    command.Parameters.Add(param);

    await using var reader = await command.ExecuteReaderAsync();

    while (await reader.ReadAsync())
    {
        // write results to console logs
        Console.WriteLine("{0} {1} {2} {3}", "Score: " + reader.GetDouble(0), "Text: " + reader.GetString(1), "Summary: " + reader.GetString(2), "Distance: " + reader.GetDouble(3));
        Console.WriteLine();

        // add results to chat history
        chatHistory.AddSystemMessage(reader.GetString(1) + ", " + reader.GetString(2));
    }
}
catch (SqlException e)
{
    Console.WriteLine($"SQL Error: {e.Message}");
}
catch (Exception e)
{
    Console.WriteLine(e.ToString());
}

Console.WriteLine("Done");
```

The SQL query itself is using a hybrid search, which executes the stored procedure you set up to create embeddings and uses SQL to filter your results. In this example, you assign scores to the results, order the output to grab the best results, and use them as grounded context to generate a response.

### Secure your data with managed identity

Azure SQL can use managed identity with Microsoft Entra to secure your SQL resource by configuring passwordless authentication. Use the following steps to configure a passwordless connection string to use in your application.

1. In the Azure portal, go to your Azure SQL server resource and select **Settings** > **Microsoft Entra ID** from the left navigation menu.
1. Select **Set admin**, search for and select yourself, and then select **Save**. Entra ID is now set up on your SQL server, and accepts Entra ID authentication.
1. Go to your database resource, select **Settings** > **Connection string** from the left navigation menu, and copy the **ADO.NET (Microsoft Entra passwordless authentication)** connection string.
1. Add the connection string to the place in your code that uses your connection string.

You can now test your application locally with your passwordless connection string.

### Grant database access to App Service 

Before you can use your web app to call your Azure SQL database using managed identity, you must grant your database access to your app.

1. In the Azure portal, go to your web app and select **Settings** > **Identity** from the left navigation menu.
1. On the **System assigned** tab, set **Status** to **On** if not already set, and then select **Save**.
1. Go to your database resource and select **Query editor** from the left navigation menu. Sign in to your database if necessary.
1. In the **Query editor**, run the following SQL commands that create the web app as a user and assign it the necessary role memberships. Replace `<your-app-name>` with your web app's name.

   ```sql
   -- Create member, alter roles to your database
   CREATE USER "<your-app-name>" FROM EXTERNAL PROVIDER;
   ALTER ROLE db_datareader ADD MEMBER "<your-app-name>";
   ALTER ROLE db_datawriter ADD MEMBER "<your-app-name>";
   ALTER ROLE db_ddladmin ADD MEMBER "<your-app-name>";
   GO
   ```

1. Next, grant the web app access to use the stored procedure and Azure OpenAI endpoint.

   ```sql
   -- Grant access to use stored procedure
   GRANT EXECUTE ON OBJECT::[dbo].[GET_EMBEDDINGS]  
     TO "<your-app-name>"  
   GO
   
   -- Grant access to use Azure OpenAI endpoint in stored procedure
   GRANT EXECUTE ANY EXTERNAL ENDPOINT TO "<your-app-name>";
   GO
   ```

Your Azure SQL database is now secure.

## 7. Deploy the app

You can now deploy your application to Azure App Service. If you want to deploy the app using an `azd` template, see [Deploy with Azure Developer CLI](https://github.com/Azure-Samples/blazor-azure-sql-vector-search?tab=readme-ov-file#deploy-with-azure-developer-cli) in the [Azure-Samples/blazor-azure-sql-vector-search](https://github.com/Azure-Samples/blazor-azure-sql-vector-search) repo.

## Complete example

The following code shows the full added *OpenAI.razor* page.

```csharp
@page "/openai"
@rendermode InteractiveServer
@inject Microsoft.Extensions.Configuration.IConfiguration _config

<PageTitle>OpenAI</PageTitle>

<h3>OpenAI input query: </h3>
<input class="col-sm-4" @bind="userMessage" />
<button class="btn btn-primary" @onclick="SemanticKernelClient">Send Request</button>

<br />
<br />

<h4>Server response:</h4> <p>@serverResponse</p>

@code {

    @using Microsoft.SemanticKernel;
    @using Microsoft.SemanticKernel.ChatCompletion;
    @using Microsoft.Data.SqlClient;

    private string? userMessage;
    private string? serverResponse;

    private async Task SemanticKernelClient()
    {
        // App settings
        string deploymentName = _config["DEPLOYMENT_NAME"];
        string endpoint = _config["ENDPOINT"];
        string apiKey = _config["API_KEY"];
        string modelId = _config["MODEL_ID"];

        // Semantic Kernel builder
        var builder = Kernel.CreateBuilder();

        // Chat completion service
        builder.Services.AddAzureOpenAIChatCompletion(
            deploymentName: deploymentName,
            endpoint: endpoint,
            apiKey: apiKey,
            modelId: modelId
        );

        var kernel = builder.Build();

        // Create prompt template
        var chat = kernel.CreateFunctionFromPrompt(
            @"{{$history}}
            User: {{$request}}
            Assistant: ");

        ChatHistory chatHistory = new("""You are a helpful assistant that answers questions about my data""");

        #region Azure SQL
        // Database connection string
        var connectionString = _config["AZURE_SQL_CONNECTIONSTRING"];

        try
        {
            await using var connection = new SqlConnection(connectionString);
            Console.WriteLine("\nQuery results:");
    
            await connection.OpenAsync();

            // Hybrid search query
            var sql =
                    @"DECLARE @e VECTOR(1536);
                    EXEC dbo.GET_EMBEDDINGS @model = 'text-embedding-ada-002', @text = '@userMessage', @embedding = @e OUTPUT;

                         -- Comprehensive query with multiple filters.
                    SELECT TOP(5)
                        f.Score,
                        f.Summary,
                        f.Text,
                        VECTOR_DISTANCE('cosine', @e, VectorBinary) AS Distance,
                        CASE
                            WHEN LEN(f.Text) > 100 THEN 'Detailed Review'
                            ELSE 'Short Review'
                        END AS ReviewLength,
                        CASE
                            WHEN f.Score >= 4 THEN 'High Score'
                            WHEN f.Score BETWEEN 2 AND 3 THEN 'Medium Score'
                            ELSE 'Low Score'
                        END AS ScoreCategory
                    FROM finefoodembeddings10k$ f
                    WHERE
                        f.UserId NOT LIKE 'Anonymous%' -- User-based filter to exclude anonymous users
                        AND f.Score >= 4 -- Score threshold filter
                        AND LEN(f.Text) > 50 -- Text length filter for detailed reviews
                        AND (f.Text LIKE '%juice%') -- Inclusion of specific words
                    ORDER BY
                        Distance,  -- Order by distance
                        f.Score DESC, -- Secondary order by review score
                        ReviewLength DESC; -- Tertiary order by review length
                ";

            // Set SQL Parameter to pass in user message
            SqlParameter param = new SqlParameter();
            param.ParameterName = "@userMessage";
            param.Value = userMessage;

            await using var command = new SqlCommand(sql, connection);

            // add parameter to SqlCommand
            command.Parameters.Add(param);

            await using var reader = await command.ExecuteReaderAsync();

            while (await reader.ReadAsync())
            {
                // write results to console logs
                Console.WriteLine("{0} {1} {2} {3}", "Score: " + reader.GetDouble(0), "Text: " + reader.GetString(1), "Summary: " + reader.GetString(2), "Distance: " + reader.GetDouble(3));
                Console.WriteLine();

                // add results to chat history
                chatHistory.AddSystemMessage(reader.GetString(1) + ", " + reader.GetString(2));

            }
        }
        catch (SqlException e)
        {
            Console.WriteLine($"SQL Error: {e.Message}");
        }
        catch (Exception e)
        {
            Console.WriteLine(e.ToString());
        }

        Console.WriteLine("Done");
        #endregion

        var chatResult = kernel.InvokeStreamingAsync<StreamingChatMessageContent>(
                chat,
                new()
                    {
                        { "request", userMessage },
                        { "history", string.Join("\n", chatHistory.Select(x => x.Role + ": " + x.Content)) }
                    }
            );

        string message = "";
        await foreach (var chunk in chatResult)
        {
            message += chunk;
        }

        // Append messages to chat history
        chatHistory.AddUserMessage(userMessage!);
        chatHistory.AddAssistantMessage(message);

        serverResponse = message;

    }
}

```

## Related content

- [`Announcing EAP for Vector Support in Azure SQL Database`](https://devblogs.microsoft.com/azure-sql/announcing-eap-native-vector-support-in-azure-sql-database/)
- [Tutorial: Build a chatbot with Azure App Service and Azure OpenAI (.NET)](tutorial-ai-openai-chatbot-dotnet.md)
- [Clone and deploy a .NET 9 Blazor chat app connected to Azure OpenAI](https://github.com/Azure-Samples/blazor-azure-sql-vector-search)
- [`Azure SQL DB vectorizer`](https://github.com/Azure-Samples/azure-sql-db-vectorizer)
