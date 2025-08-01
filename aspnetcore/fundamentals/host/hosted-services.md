---
title: Background tasks with hosted services in ASP.NET Core
author: tdykstra
description: Learn how to implement background tasks with hosted services in ASP.NET Core.
monikerRange: '>= aspnetcore-3.1'
ms.author: tdykstra
ms.custom: mvc
ms.date: 05/28/2025
uid: fundamentals/host/hosted-services
---
# Background tasks with hosted services in ASP.NET Core

By [Jeow Li Huan](https://github.com/huan086)

[!INCLUDE[](~/includes/not-latest-version.md)]

:::moniker range=">= aspnetcore-8.0"

In ASP.NET Core, background tasks can be implemented as *hosted services*. A hosted service is a class with background task logic that implements the <xref:Microsoft.Extensions.Hosting.IHostedService> interface. This article provides three hosted service examples:

* Background task that runs on a timer.
* Hosted service that activates a [scoped service](xref:fundamentals/dependency-injection#service-lifetimes). The scoped service can use [dependency injection (DI)](xref:fundamentals/dependency-injection).
* Queued background tasks that run sequentially.

## Worker Service template

The ASP.NET Core Worker Service template provides a starting point for writing long running service apps. An app created from the Worker Service template specifies the Worker SDK in its project file:

```xml
<Project Sdk="Microsoft.NET.Sdk.Worker">
```

To use the template as a basis for a hosted services app:

[!INCLUDE[](~/includes/worker-template-instructions-net6.md)]

## Package

An app based on the Worker Service template uses the `Microsoft.NET.Sdk.Worker` SDK and has an explicit package reference to the [Microsoft.Extensions.Hosting](https://www.nuget.org/packages/Microsoft.Extensions.Hosting) package. For example, see the sample app's project file (`BackgroundTasksSample.csproj`).

For web apps that use the `Microsoft.NET.Sdk.Web` SDK, the [Microsoft.Extensions.Hosting](https://www.nuget.org/packages/Microsoft.Extensions.Hosting) package is referenced implicitly from the shared framework. An explicit package reference in the app's project file isn't required.

## IHostedService interface

The <xref:Microsoft.Extensions.Hosting.IHostedService> interface defines two methods for objects that are managed by the host:

* [StartAsync(CancellationToken)](xref:Microsoft.Extensions.Hosting.IHostedService.StartAsync%2A)
* [StopAsync(CancellationToken)](xref:Microsoft.Extensions.Hosting.IHostedService.StopAsync%2A)

### `StartAsync`

[StartAsync(CancellationToken)](xref:Microsoft.Extensions.Hosting.IHostedService.StartAsync%2A) contains the logic to start the background task. `StartAsync` is called *before*:

* The app's request processing pipeline is configured.
* The server is started and [IApplicationLifetime.ApplicationStarted](xref:Microsoft.AspNetCore.Hosting.IApplicationLifetime.ApplicationStarted%2A) is triggered.

`StartAsync` should be limited to short running tasks because hosted services are run sequentially, and no further services are started until `StartAsync` runs to completion.

### `StopAsync`

* [StopAsync(CancellationToken)](xref:Microsoft.Extensions.Hosting.IHostedService.StopAsync%2A) is triggered when the host is performing a graceful shutdown. `StopAsync` contains the logic to end the background task. Implement <xref:System.IDisposable> and [finalizers (destructors)](/dotnet/csharp/programming-guide/classes-and-structs/destructors) to dispose of any unmanaged resources.

The cancellation token has a default 30 second timeout to indicate that the shutdown process should no longer be graceful. When cancellation is requested on the token:

* Any remaining background operations that the app is performing should be aborted.
* Any methods called in `StopAsync` should return promptly.

However, tasks aren't abandoned after cancellation is requested&mdash;the caller awaits all tasks to complete.

If the app shuts down unexpectedly (for example, the app's process fails), `StopAsync` might not be called. Therefore, any methods called or operations conducted in `StopAsync` might not occur.

To extend the default 30 second shutdown timeout, set:

* <xref:Microsoft.Extensions.Hosting.HostOptions.ShutdownTimeout%2A> when using Generic Host. For more information, see <xref:fundamentals/host/generic-host#shutdowntimeout>.
* Shutdown timeout host configuration setting when using Web Host. For more information, see <xref:fundamentals/host/web-host#shutdown-timeout>.

The hosted service is activated once at app startup and gracefully shut down at app shutdown. If an error is thrown during background task execution, `Dispose` should be called even if `StopAsync` isn't called.

## BackgroundService base class

<xref:Microsoft.Extensions.Hosting.BackgroundService> is a base class for implementing a long running <xref:Microsoft.Extensions.Hosting.IHostedService>.

[ExecuteAsync(CancellationToken)](xref:Microsoft.Extensions.Hosting.BackgroundService.ExecuteAsync%2A) is called to run the background service. The implementation returns a <xref:System.Threading.Tasks.Task> that represents the entire lifetime of the background service. No further services are started until [ExecuteAsync becomes asynchronous](https://github.com/dotnet/extensions/issues/2149), such as by calling `await`. Avoid performing long, blocking initialization work in `ExecuteAsync`. The host blocks in [StopAsync(CancellationToken)](xref:Microsoft.Extensions.Hosting.BackgroundService.StopAsync%2A) waiting for `ExecuteAsync` to complete.

The cancellation token is triggered when [IHostedService.StopAsync](xref:Microsoft.Extensions.Hosting.IHostedService.StopAsync%2A) is called. Your implementation of `ExecuteAsync` should finish promptly when the cancellation token is fired in order to gracefully shut down the service. Otherwise, the service ungracefully shuts down at the shutdown timeout. For more information, see the [IHostedService interface](#ihostedservice-interface) section.

For more information, see the [BackgroundService](https://github.com/dotnet/runtime/blob/main/src/libraries/Microsoft.Extensions.Hosting.Abstractions/src/BackgroundService.cs) source code.

## Timed background tasks

A timed background task makes use of the [System.Threading.Timer](xref:System.Threading.Timer) class. The timer triggers the task's `DoWork` method. The timer is disabled on `StopAsync` and disposed when the service container is disposed on `Dispose`:

:::code language="csharp" source="~/fundamentals/host/hosted-services/samples/6.0/BackgroundTasksSample/Services/TimedHostedService.cs" id="snippet1" highlight="16-17,34,41":::

The <xref:System.Threading.Timer> doesn't wait for previous executions of `DoWork` to finish, so the approach shown might not be suitable for every scenario. [Interlocked.Increment](xref:System.Threading.Interlocked.Increment%2A) is used to increment the execution counter as an atomic operation, which ensures that multiple threads don't update `executionCount` concurrently.

The service is registered in `IHostBuilder.ConfigureServices` (`Program.cs`) with the `AddHostedService` extension method:

:::code language="csharp" source="~/fundamentals/host/hosted-services/samples/6.0/BackgroundTasksSample/Program.cs" id="snippet1":::

## Consuming a scoped service in a background task

To use [scoped services](xref:fundamentals/dependency-injection#service-lifetimes) within a [BackgroundService](#backgroundservice-base-class), create a scope. No scope is created for a hosted service by default.

The scoped background task service contains the background task's logic. In the following example:

* The service is asynchronous. The `DoWork` method returns a `Task`. For demonstration purposes, a delay of ten seconds is awaited in the `DoWork` method.
* An <xref:Microsoft.Extensions.Logging.ILogger> is injected into the service.

:::code language="csharp" source="~/fundamentals/host/hosted-services/samples/6.0/BackgroundTasksSample/Services/ScopedProcessingService.cs" id="snippet1":::

The hosted service creates a scope to resolve the scoped background task service to call its `DoWork` method. `DoWork` returns a `Task`, which is awaited in `ExecuteAsync`:

:::code language="csharp" source="~/fundamentals/host/hosted-services/samples/6.0/BackgroundTasksSample/Services/ConsumeScopedServiceHostedService.cs" id="snippet1" highlight="19,22-35":::

The services are registered in `IHostBuilder.ConfigureServices` (`Program.cs`). The hosted service is registered with the `AddHostedService` extension method:

:::code language="csharp" source="~/fundamentals/host/hosted-services/samples/3.x/BackgroundTasksSample/Program.cs" id="snippet2":::

## Queued background tasks

A background task queue is based on the .NET Framework 4.x <xref:System.Web.Hosting.HostingEnvironment.QueueBackgroundWorkItem%2A>:

:::code language="csharp" source="~/fundamentals/host/hosted-services/samples/6.0/BackgroundTasksSample/Services/BackgroundTaskQueue.cs" id="snippet1":::

In the following `QueueHostedService` example:

* The `BackgroundProcessing` method returns a `Task`, which is awaited in `ExecuteAsync`.
* Background tasks in the queue are dequeued and executed in `BackgroundProcessing`.
* Work items are awaited before the service stops in `StopAsync`.

:::code language="csharp" source="~/fundamentals/host/hosted-services/samples/6.0/BackgroundTasksSample/Services/QueuedHostedService.cs" id="snippet1" highlight="28-29,33":::

A `MonitorLoop` service handles enqueuing tasks for the hosted service whenever the `w` key is selected on an input device:

* The `IBackgroundTaskQueue` is injected into the `MonitorLoop` service.
* `IBackgroundTaskQueue.QueueBackgroundWorkItem` is called to enqueue a work item.
* The work item simulates a long-running background task:
  * Three 5-second delays are executed (`Task.Delay`).
  * A `try-catch` statement traps <xref:System.OperationCanceledException> if the task is cancelled.

:::code language="csharp" source="~/fundamentals/host/hosted-services/samples/6.0/BackgroundTasksSample/Services/MonitorLoop.cs" id="snippet_Monitor" highlight="7,33":::

The services are registered in `IHostBuilder.ConfigureServices` (`Program.cs`). The hosted service is registered with the `AddHostedService` extension method:

:::code language="csharp" source="~/fundamentals/host/hosted-services/samples/6.0/BackgroundTasksSample/Program.cs" id="snippet3":::

`MonitorLoop` is started in `Program.cs`:

:::code language="csharp" source="~/fundamentals/host/hosted-services/samples/6.0/BackgroundTasksSample/Program.cs" id="snippet4":::

## Asynchronous timed background task

The following code creates an asynchronous timed background task:

:::code language="csharp" source="~/../AspNetCore.Docs.Samples/fundamentals/host/TimedBackgroundTasks/TimedHostedService.cs":::

## Native AOT

The Worker Service templates support [.NET native ahead-of-time (AOT)](/dotnet/core/deploying/native-aot/) with the `--aot` flag:

# [Visual Studio](#tab/visual-studio)

1. Create a new project.
1. Select **Worker Service**. Select **Next**.
1. Provide a project name in the **Project name** field or accept the default project name.  Select **Next**.
1. In the **Additional information** dialog:
  1. Choose a **Framework**.
  1. Check the **Enable Native AOT publish** checkbox.
  1. Select **Create**.

# [.NET CLI](#tab/net-cli)

Use the Worker Service (`worker`) template with the [dotnet new](/dotnet/core/tools/dotnet-new) command from a command shell with the AOT option:

```dotnetcli
dotnet new worker -o WorkerWithAot --aot
```

---

The AOT option adds `<PublishAot>true</PublishAot>` to the project file:

```diff

<Project Sdk="Microsoft.NET.Sdk.Worker">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <InvariantGlobalization>true</InvariantGlobalization>
+   <PublishAot>true</PublishAot>
    <UserSecretsId>dotnet-WorkerWithAot-e94b2</UserSecretsId>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Hosting" Version="8.0.0-preview.4.23259.5" />
  </ItemGroup>
</Project>
```

## Additional resources

* [Background services unit tests on GitHub](https://github.com/dotnet/runtime/blob/main/src/libraries/Microsoft.Extensions.Hosting/tests/UnitTests/BackgroundServiceTests.cs).
* [View or download sample code](https://github.com/dotnet/AspNetCore.Docs/tree/main/aspnetcore/fundamentals/host/hosted-services/samples/) ([how to download](xref:index#how-to-download-a-sample))
* [Implement background tasks in microservices with IHostedService and the BackgroundService class](/dotnet/standard/microservices-architecture/multi-container-microservice-net-applications/background-tasks-with-ihostedservice)
* [Run background tasks with WebJobs in Azure App Service](/azure/app-service/webjobs-create)
* <xref:System.Threading.Timer>

:::moniker-end

[!INCLUDE[](~/fundamentals/host/hosted-services/includes/hosted-services67.md)]
[!INCLUDE[](~/fundamentals/host/hosted-services/includes/hosted-services5.md)]
