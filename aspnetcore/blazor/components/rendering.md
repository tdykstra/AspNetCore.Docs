---
title: ASP.NET Core Razor component rendering
author: guardrex
description: Learn about Razor component rendering in ASP.NET Core Blazor apps, including when to manually trigger a component to render.
monikerRange: '>= aspnetcore-3.1'
ms.author: wpickett
ms.custom: mvc
ms.date: 11/12/2024
uid: blazor/components/rendering
---
# ASP.NET Core Razor component rendering

[!INCLUDE[](~/includes/not-latest-version.md)]

This article explains Razor component rendering in ASP.NET Core Blazor apps, including when to call <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A> to manually trigger a component to render.

## Rendering conventions for `ComponentBase`

Components *must* render when they're first added to the component hierarchy by a parent component. This is the only time that a component must render. Components *may* render at other times according to their own logic and conventions.

Razor components inherit from the <xref:Microsoft.AspNetCore.Components.ComponentBase> base class, which contains logic to trigger rerendering at the following times:

* After applying an updated set of [parameters](xref:blazor/components/data-binding#binding-with-component-parameters) from a parent component.
* After applying an updated value for a [cascading parameter](xref:blazor/components/cascading-values-and-parameters).
* After notification of an event and invoking one of its own [event handlers](xref:blazor/components/event-handling).
* After a call to its own <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A> method (see <xref:blazor/components/lifecycle#state-changes-statehaschanged>). For guidance on how to prevent overwriting child component parameters when <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A> is called in a parent component, see <xref:blazor/components/overwriting-parameters>.

Components inherited from <xref:Microsoft.AspNetCore.Components.ComponentBase> skip rerenders due to parameter updates if either of the following are true:

* All of the parameters are from a set of known types&dagger; or any [primitive type](/dotnet/api/system.type.isprimitive) that hasn't changed since the previous set of parameters were set.

  &dagger;The Blazor framework uses a set of built-in rules and explicit parameter type checks for change detection. These rules and the types are subject to change at any time. For more information, see the [`ChangeDetection` API in the ASP.NET Core reference source](https://github.com/dotnet/aspnetcore/blob/main/src/Components/Components/src/ChangeDetection.cs).
  
  [!INCLUDE[](~/includes/aspnetcore-repo-ref-source-links.md)]

* The override of the component's [`ShouldRender` method](#suppress-ui-refreshing-shouldrender) returns `false` (the default `ComponentBase` implementation always returns `true`).

## Control the rendering flow

In most cases, <xref:Microsoft.AspNetCore.Components.ComponentBase> conventions result in the correct subset of component rerenders after an event occurs. Developers aren't usually required to provide manual logic to tell the framework which components to rerender and when to rerender them. The overall effect of the framework's conventions is that the component receiving an event rerenders itself, which recursively triggers rerendering of descendant components whose parameter values may have changed.

For more information on the performance implications of the framework's conventions and how to optimize an app's component hierarchy for rendering, see <xref:blazor/performance/rendering>.

::: moniker range=">= aspnetcore-8.0"

## Streaming rendering

Use *streaming rendering* with [static server-side rendering (static SSR)](xref:blazor/components/render-modes) or prerendering to stream content updates on the response stream and improve the user experience for components that perform long-running asynchronous tasks to fully render.

For example, consider a component that makes a long-running database query or web API call to render data when the page loads. Normally, asynchronous tasks executed as part of rendering a server-side component must complete before the rendered response is sent, which can delay loading the page. Any significant delay in rendering the page harms the user experience. To improve the user experience, streaming rendering initially renders the entire page quickly with placeholder content while asynchronous operations execute. After the operations are complete, the updated content is sent to the client on the same response connection and patched into the DOM.

Streaming rendering requires the server to avoid buffering the output. The response data must flow to the client as the data is generated. For hosts that enforce buffering, streaming rendering degrades gracefully, and the page loads without streaming rendering.

To stream content updates when using static server-side rendering (static SSR) or prerendering, apply the [`[StreamRendering]` attribute](xref:Microsoft.AspNetCore.Components.StreamRenderingAttribute) in .NET 9 or later (use `[StreamRendering(true)]` in .NET 8) to the component. Streaming rendering must be explicitly enabled because streamed updates may cause content on the page to shift. Components without the attribute automatically adopt streaming rendering if the parent component uses the feature. Pass `false` to the attribute in a child component to disable the feature at that point and further down the component subtree. The attribute is functional when applied to components supplied by a [Razor class library](xref:blazor/components/class-libraries).

:::moniker-end

:::moniker range=">= aspnetcore-10.0"

If [enhanced navigation](xref:blazor/fundamentals/routing#enhanced-navigation-and-form-handling) is active, streaming rendering renders [Not Found responses](xref:blazor/fundamentals/routing#not-found-responses) without reloading the page. When enhanced navigation is blocked, the framework redirects to Not Found content with a page refresh. 

Streaming rendering can only render components that have a route, such as a [`NotFoundPage` assignment](xref:blazor/fundamentals/routing#not-found-responses) (`NotFoundPage="..."`) or a [Status Code Pages Re-execution Middleware page assignment](xref:fundamentals/error-handling#usestatuscodepageswithreexecute) (<xref:Microsoft.AspNetCore.Builder.StatusCodePagesExtensions.UseStatusCodePagesWithReExecute%2A>). The Not Found render fragment (`<NotFound>...</NotFound>`) and the `DefaultNotFound` 404 content ("`Not found`" plain text) don't have routes, so they can't be used during streaming rendering.

Streaming `NavigationManager.NotFound` content rendering uses (in order):

* A `NotFoundPage` passed to the `Router` component, if present.
* A Status Code Pages Re-execution Middleware page, if configured.
* No action if neither of the preceding approaches is adopted.

Non-streaming `NavigationManager.NotFound` content rendering uses (in order):

* A `NotFoundPage` passed to the `Router` component, if present.
* Not Found render fragment content, if present. *Not recommended in .NET 10 or later.*
* `DefaultNotFound` 404 content ("`Not found`" plain text).

[Status Code Pages Re-execution Middleware](xref:fundamentals/error-handling#usestatuscodepageswithreexecute) with <xref:Microsoft.AspNetCore.Builder.StatusCodePagesExtensions.UseStatusCodePagesWithReExecute%2A> takes precedence for browser-based address routing problems, such as an incorrect URL typed into the browser's address bar or selecting a link that has no endpoint in the app.

:::moniker-end

:::moniker range=">= aspnetcore-8.0"

The following example is based on the `Weather` component in an app created from the [Blazor Web App project template](xref:blazor/project-structure#blazor-web-app). The call to <xref:System.Threading.Tasks.Task.Delay%2A?displayProperty=nameWithType> simulates retrieving weather data asynchronously. The component initially renders placeholder content ("`Loading...`") without waiting for the asynchronous delay to complete. When the asynchronous delay completes and the weather data content is generated, the content is streamed to the response and patched into the weather forecast table.

`Weather.razor`:

```razor
@page "/weather"
@attribute [StreamRendering]

...

@if (forecasts == null)
{
    <p><em>Loading...</em></p>
}
else
{
    <table class="table">
        ...
        <tbody>
            @foreach (var forecast in forecasts)
            {
                <tr>
                    <td>@forecast.Date.ToShortDateString()</td>
                    <td>@forecast.TemperatureC</td>
                    <td>@forecast.TemperatureF</td>
                    <td>@forecast.Summary</td>
                </tr>
            }
        </tbody>
    </table>
}

@code {
    ...

    private WeatherForecast[]? forecasts;

    protected override async Task OnInitializedAsync()
    {
        await Task.Delay(500);

        ...

        forecasts = ...
    }
}
```

:::moniker-end

## Suppress UI refreshing (`ShouldRender`)

<xref:Microsoft.AspNetCore.Components.ComponentBase.ShouldRender%2A> is called each time a component is rendered. Override <xref:Microsoft.AspNetCore.Components.ComponentBase.ShouldRender%2A> to manage UI refreshing. If the implementation returns `true`, the UI is refreshed.

Even if <xref:Microsoft.AspNetCore.Components.ComponentBase.ShouldRender%2A> is overridden, the component is always initially rendered.

`ControlRender.razor`:

:::moniker range=">= aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/9.0/BlazorSample_BlazorWebApp/Components/Pages/ControlRender.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/Components/Pages/ControlRender.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-7.0 < aspnetcore-8.0"

:::code language="razor" source="~/../blazor-samples/7.0/BlazorSample_WebAssembly/Pages/rendering/ControlRender.razor":::

::: moniker-end

::: moniker range=">= aspnetcore-6.0 < aspnetcore-7.0"

:::code language="razor" source="~/../blazor-samples/6.0/BlazorSample_WebAssembly/Pages/rendering/ControlRender.razor":::

::: moniker-end

::: moniker range=">= aspnetcore-5.0 < aspnetcore-6.0"

:::code language="razor" source="~/../blazor-samples/5.0/BlazorSample_WebAssembly/Pages/rendering/ControlRender.razor":::

::: moniker-end

::: moniker range="< aspnetcore-5.0"

:::code language="razor" source="~/../blazor-samples/3.1/BlazorSample_WebAssembly/Pages/rendering/ControlRender.razor":::

::: moniker-end

For more information on performance best practices pertaining to <xref:Microsoft.AspNetCore.Components.ComponentBase.ShouldRender%2A>, see <xref:blazor/performance/rendering#avoid-unnecessary-rendering-of-component-subtrees>.

## `StateHasChanged`

Calling <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A> enqueues a rerender to occur when the app's main thread is free.

Components are enqueued for rendering, and they aren't enqueued again if there's already a pending rerender. If a component calls <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A> five times in a row in a loop, the component only renders once. This behavior is encoded in <xref:Microsoft.AspNetCore.Components.ComponentBase>, which checks first if it has queued a rerender before enqueuing an additional one.

A component can render multiple times during the same cycle, which commonly occurs when a component has children that interact with each other:

* A parent component renders several children.
* Child components render and trigger an update on the parent.
* A parent component rerenders with new state.

This design allows for <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A> to be called when necessary without the risk of introducing unnecessary rendering. You can always take control of this behavior in individual components by implementing <xref:Microsoft.AspNetCore.Components.IComponent> directly and manually handling when the component renders.

Consider the following `IncrementCount` method that increments a count, calls <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A>, and increments the count again:

```csharp
private void IncrementCount()
{
    currentCount++;
    StateHasChanged();
    currentCount++;
}
```

Stepping through the code in the debugger, you might think that the count updates in the UI for the first `currentCount++` execution immediately after <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A> is called. However, the UI doesn't show an updated count at that point due to the synchronous processing taking place for this method's execution. There's no opportunity for the renderer to render the component until after the event handler is finished. The UI displays increases for both `currentCount++` executions *in a single render*.

If you await something between the `currentCount++` lines, the awaited call gives the renderer a chance to render. This has led to some developers calling <xref:System.Threading.Tasks.Task.Delay%2A> with a one millisecond delay in their components to allow a render to occur, but we don't recommend arbitrarily slowing down an app to enqueue a render.

The best approach is to await <xref:System.Threading.Tasks.Task.Yield%2A?displayProperty=nameWithType>, which forces the component to process code asynchronously and render during the current batch with a second render in a separate batch after the yielded task runs the continuation.

Consider the following revised `IncrementCount` method, which updates the UI twice because the render enqueued by <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A> is performed when the task is yielded with the call to <xref:System.Threading.Tasks.Task.Yield%2A?displayProperty=nameWithType>:

```csharp
private async Task IncrementCount()
{
    currentCount++;
    StateHasChanged();
    await Task.Yield();
    currentCount++;
}
```

Be careful not to call <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A> unnecessarily, which is a common mistake that imposes unnecessary rendering costs. Code shouldn't need to call <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A> when:

* Routinely handling events, whether synchronously or asynchronously, since <xref:Microsoft.AspNetCore.Components.ComponentBase> triggers a render for most routine event handlers.
* Implementing typical lifecycle logic, such as [`OnInitialized`](xref:blazor/components/lifecycle#component-initialization-oninitializedasync) or [`OnParametersSetAsync`](xref:blazor/components/lifecycle#after-parameters-are-set-onparameterssetasync), whether synchronously or asynchronously, since <xref:Microsoft.AspNetCore.Components.ComponentBase> triggers a render for typical lifecycle events.

However, it might make sense to call <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A> in the cases described in the following sections of this article:

* [An asynchronous handler involves multiple asynchronous phases](#an-asynchronous-handler-involves-multiple-asynchronous-phases)
* [Receiving a call from something external to the Blazor rendering and event handling system](#receiving-a-call-from-something-external-to-the-blazor-rendering-and-event-handling-system)
* [To render component outside the subtree that is rerendered by a particular event](#to-render-a-component-outside-the-subtree-thats-rerendered-by-a-particular-event)

### An asynchronous handler involves multiple asynchronous phases

Due to the way that tasks are defined in .NET, a receiver of a <xref:System.Threading.Tasks.Task> can only observe its final completion, not intermediate asynchronous states. Therefore, <xref:Microsoft.AspNetCore.Components.ComponentBase> can only trigger rerendering when the <xref:System.Threading.Tasks.Task> is first returned and when the <xref:System.Threading.Tasks.Task> finally completes. The framework can't know to rerender a component at other intermediate points, such as when an <xref:System.Collections.Generic.IAsyncEnumerable%601> [returns data in a series of intermediate `Task`s](https://github.com/dotnet/aspnetcore/issues/43098#issuecomment-1206224427). If you want to rerender at intermediate points, call <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A> at those points.

Consider the following `CounterState1` component, which updates the count four times each time the `IncrementCount` method executes:

* Automatic renders occur after the first and last increments of `currentCount`.
* Manual renders are triggered by calls to <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A> when the framework doesn't automatically trigger rerenders at intermediate processing points where `currentCount` is incremented.

`CounterState1.razor`:

:::moniker range=">= aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/9.0/BlazorSample_BlazorWebApp/Components/Pages/CounterState1.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/Components/Pages/CounterState1.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-7.0 < aspnetcore-8.0"

:::code language="razor" source="~/../blazor-samples/7.0/BlazorSample_WebAssembly/Pages/rendering/CounterState1.razor":::

::: moniker-end

::: moniker range=">= aspnetcore-6.0 < aspnetcore-7.0"

:::code language="razor" source="~/../blazor-samples/6.0/BlazorSample_WebAssembly/Pages/rendering/CounterState1.razor":::

::: moniker-end

::: moniker range=">= aspnetcore-5.0 < aspnetcore-6.0"

:::code language="razor" source="~/../blazor-samples/5.0/BlazorSample_WebAssembly/Pages/rendering/CounterState1.razor":::

::: moniker-end

::: moniker range="< aspnetcore-5.0"

:::code language="razor" source="~/../blazor-samples/3.1/BlazorSample_WebAssembly/Pages/rendering/CounterState1.razor":::

::: moniker-end

### Receiving a call from something external to the Blazor rendering and event handling system

<xref:Microsoft.AspNetCore.Components.ComponentBase> only knows about its own lifecycle methods and Blazor-triggered events. <xref:Microsoft.AspNetCore.Components.ComponentBase> doesn't know about other events that may occur in code. For example, any C# events raised by a custom data store are unknown to Blazor. In order for such events to trigger rerendering to display updated values in the UI, call <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A>.

Consider the following `CounterState2` component that uses <xref:System.Timers.Timer?displayProperty=fullName> to update a count at a regular interval and calls <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A> to update the UI:

* `OnTimerCallback` runs outside of any Blazor-managed rendering flow or event notification. Therefore, `OnTimerCallback` must call <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A> because Blazor isn't aware of the changes to `currentCount` in the callback.
* The component implements <xref:System.IDisposable>, where the <xref:System.Timers.Timer> is disposed when the framework calls the `Dispose` method. For more information, see <xref:blazor/components/component-disposal>.

Because the callback is invoked outside of Blazor's synchronization context, the component must wrap the logic of `OnTimerCallback` in <xref:Microsoft.AspNetCore.Components.ComponentBase.InvokeAsync%2A?displayProperty=nameWithType> to move it onto the renderer's synchronization context. This is equivalent to marshalling to the UI thread in other UI frameworks. <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A> can only be called from the renderer's synchronization context and throws an exception otherwise:

> :::no-loc text="System.InvalidOperationException: 'The current thread is not associated with the Dispatcher. Use InvokeAsync() to switch execution to the Dispatcher when triggering rendering or component state.'":::

`CounterState2.razor`:

:::moniker range=">= aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/9.0/BlazorSample_BlazorWebApp/Components/Pages/CounterState2.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/Components/Pages/CounterState2.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-7.0 < aspnetcore-8.0"

:::code language="razor" source="~/../blazor-samples/7.0/BlazorSample_WebAssembly/Pages/rendering/CounterState2.razor":::

::: moniker-end

::: moniker range=">= aspnetcore-6.0 < aspnetcore-7.0"

:::code language="razor" source="~/../blazor-samples/6.0/BlazorSample_WebAssembly/Pages/rendering/CounterState2.razor":::

::: moniker-end

::: moniker range=">= aspnetcore-5.0 < aspnetcore-6.0"

:::code language="razor" source="~/../blazor-samples/5.0/BlazorSample_WebAssembly/Pages/rendering/CounterState2.razor":::

::: moniker-end

::: moniker range="< aspnetcore-5.0"

:::code language="razor" source="~/../blazor-samples/3.1/BlazorSample_WebAssembly/Pages/rendering/CounterState2.razor":::

::: moniker-end

### To render a component outside the subtree that's rerendered by a particular event

The UI might involve:

1. Dispatching an event to one component.
1. Changing some state.
1. Rerendering a completely different component that isn't a descendant of the component receiving the event.

One way to deal with this scenario is to provide a *state management* class, often as a dependency injection (DI) service, injected into multiple components. When one component calls a method on the state manager, the state manager raises a C# event that's then received by an independent component.

For approaches to manage state, see the following resources:

* [Bind across more than two components](xref:blazor/components/data-binding#bind-across-more-than-two-components) using data bindings.
* [Pass data across a component hierarchy](xref:blazor/components/cascading-values-and-parameters#pass-data-across-a-component-hierarchy) using cascading values and parameters.
* [Server-side in-memory state container service](xref:blazor/state-management?pivots=server#in-memory-state-container-service-server) ([client-side equivalent](xref:blazor/state-management?pivots=webassembly#in-memory-state-container-service-wasm)) section of the *State management* article.

For the state manager approach, C# events are outside the Blazor rendering pipeline. Call <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A> on other components you wish to rerender in response to the state manager's events.

The state manager approach is similar to the earlier case with <xref:System.Timers.Timer?displayProperty=fullName> in the [previous section](#receiving-a-call-from-something-external-to-the-blazor-rendering-and-event-handling-system). Since the execution call stack typically remains on the renderer's synchronization context, calling <xref:Microsoft.AspNetCore.Components.ComponentBase.InvokeAsync%2A> isn't normally required. Calling <xref:Microsoft.AspNetCore.Components.ComponentBase.InvokeAsync%2A> is only required if the logic escapes the synchronization context, such as calling <xref:System.Threading.Tasks.Task.ContinueWith%2A> on a <xref:System.Threading.Tasks.Task> or awaiting a <xref:System.Threading.Tasks.Task> with [`ConfigureAwait(false)`](xref:System.Threading.Tasks.Task.ConfigureAwait%2A). For more information, see the [Receiving a call from something external to the Blazor rendering and event handling system](#receiving-a-call-from-something-external-to-the-blazor-rendering-and-event-handling-system) section.

## WebAssembly loading progress indicator for Blazor Web Apps

<!-- UPDATE 10.0 Will be removed for a new feature in this area. 
                 Tracked by: https://github.com/dotnet/aspnetcore/issues/49056 -->

A loading progress indicator isn't present in an app created from the Blazor Web App project template. A new loading progress indicator feature is planned for a future release of .NET. In the meantime, an app can adopt custom code to create a loading progress indicator. For more information, see <xref:blazor/fundamentals/startup#client-side-loading-indicators>.
