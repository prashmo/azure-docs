---
title: Migrate from the bulk executor library to the bulk support in Azure Cosmos DB .NET V3 SDK
description: Learn how to migrate your application from using the bulk executor library to the bulk support in Azure Cosmos DB SDK V3
author: ealsur
ms.service: cosmos-db
ms.topic: conceptual
ms.date: 03/24/2020
ms.author: maquaran
---

# Migrate from the bulk executor library to the bulk support in Azure Cosmos DB .NET V3 SDK

This article describes the required steps to migrate an existing application's code that uses the [.NET bulk executor library](bulk-executor-dot-net.md) to the [bulk support](tutorial-sql-api-dotnet-bulk-import.md) feature in the latest version of the .NET SDK.

## Enable bulk support

Enable bulk support on the `CosmosClient` instance through the [AllowBulkExecution](https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.cosmosclientoptions.allowbulkexecution) configuration:

   :::code language="csharp" source="~/samples-cosmosdb-dotnet-v3/Microsoft.Azure.Cosmos.Samples/Usage/BulkExecutorMigration/Program.cs" ID="Initialization":::

## Create Tasks for each operation

Bulk support in the .NET SDK works by leveraging the [Task Parallel Library](https://docs.microsoft.com/dotnet/standard/parallel-programming/task-parallel-library-tpl) and grouping operations that occur concurrently. 

There is no single method that will take your list of documents or operations as an input parameter, but rather, you need to create a Task for each operation you want to execute in bulk.

For example, if your initial input is a list of items where each item has the following schema:

   :::code language="csharp" source="~/samples-cosmosdb-dotnet-v3/Microsoft.Azure.Cosmos.Samples/Usage/BulkExecutorMigration/Program.cs" ID="Model":::

If you want to do bulk import (similar to using BulkExecutor.BulkImportAsync), you need to have concurrent calls to `CreateItemAsync` with each item value. For example:

   :::code language="csharp" source="~/samples-cosmosdb-dotnet-v3/Microsoft.Azure.Cosmos.Samples/Usage/BulkExecutorMigration/Program.cs" ID="BulkImport":::

If you want to do bulk *update* (similar to using [BulkExecutor.BulkUpdateAsync](https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmosdb.bulkexecutor.bulkexecutor.bulkupdateasync)), you need to have concurrent calls to `ReplaceItemAsync` method after updating the item value. For example:

   :::code language="csharp" source="~/samples-cosmosdb-dotnet-v3/Microsoft.Azure.Cosmos.Samples/Usage/BulkExecutorMigration/Program.cs" ID="BulkUpdate":::

And if you want to do bulk *delete* (similar to using [BulkExecutor.BulkDeleteAsync](https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmosdb.bulkexecutor.bulkexecutor.bulkdeleteasync)), you need to have concurrent calls to `DeleteItemAsync`, with the `id` and partition key of each item. For example:

   :::code language="csharp" source="~/samples-cosmosdb-dotnet-v3/Microsoft.Azure.Cosmos.Samples/Usage/BulkExecutorMigration/Program.cs" ID="BulkDelete":::

## Capture task result state

In the previous code examples, you have created a concurrent list of tasks, and called the `CaptureOperationResponse` method on each of those tasks. This method is an extension that lets us maintain a *similar response schema* as BulkExecutor, by capturing any errors and tracking the [request units usage](request-units.md).

   :::code language="csharp" source="~/samples-cosmosdb-dotnet-v3/Microsoft.Azure.Cosmos.Samples/Usage/BulkExecutorMigration/Program.cs" ID="CaptureOperationResult":::

Where the `OperationResponse` is declared as:

   :::code language="csharp" source="~/samples-cosmosdb-dotnet-v3/Microsoft.Azure.Cosmos.Samples/Usage/BulkExecutorMigration/Program.cs" ID="OperationResult":::

## Execute operations concurrently

After the list of tasks are defined, wait until they are all complete. You can track the completion of the tasks by defining the scope of your bulk operation as shown in the following code snippet:

   :::code language="csharp" source="~/samples-cosmosdb-dotnet-v3/Microsoft.Azure.Cosmos.Samples/Usage/BulkExecutorMigration/Program.cs" ID="WhenAll":::

## Capture statistics

The previous code waits until all operations are completed and calculates the required statistics. These statistics are similar to that of the bulk executor library's [BulkImportResponse](https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmosdb.bulkexecutor.bulkimport.bulkimportresponse).

   :::code language="csharp" source="~/samples-cosmosdb-dotnet-v3/Microsoft.Azure.Cosmos.Samples/Usage/BulkExecutorMigration/Program.cs" ID="ResponseType":::

The `BulkOperationResponse` contains:

1. The total time taken to process the list of operations through bulk support.
1. The number of successful operations.
1. The total of request units consumed.
1. If there are failures, it displays a list of tuples that contain the exception and the associated item for logging and identification purpose.

## Performance improvements

As with other operations with the .NET SDK, using the stream APIs results in better performance and avoids any unnecessary serialization. 

Using stream APIs is only possible if the nature of the data you use matches that of a stream of bytes (for example, file streams). In such cases, using the `CreateItemStreamAsync`, `ReplaceItemStreamAsync`, or `DeleteItemStreamAsync` methods and working with `ResponseMessage` (instead of `ItemResponse`) increases the throughput that can be achieved.

## Next steps

* To learn more about the .NET SDK releases, see the [Azure Cosmos DB SDK](sql-api-sdk-dotnet.md) article.
* Get the complete [migration source code](https://github.com/Azure/azure-cosmos-dotnet-v3/tree/master/Microsoft.Azure.Cosmos.Samples/Usage/BulkExecutorMigration) from GitHub.
* [Additional bulk samples on GitHub](https://github.com/Azure/azure-cosmos-dotnet-v3/tree/master/Microsoft.Azure.Cosmos.Samples/Usage/BulkSupport)
