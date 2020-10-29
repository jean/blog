---
layout: posts
title: Rolling in render trees (part 2)
tags: [blazor]
---

This is the third blog post in a mini-series on the internals of rendering and components in Blazor. By internals, I mean that we’ll be doing one of my favorite things: reading code and figuring out what it does. As a preface, this blog post assumes that you’re familiar with [Blazor](https://blazor.net).

In [previous blog post](./rolling-in-render-trees/), we took a peek inside the `StateHasChanged` method and unravelled some of the concepts that exist during rendering: such as `RenderHandles` and `Dispatchers` and the existence of the render queue. We left off our discussion at the work involves in processing render tasks. The [`ProcessRenderQueue`](https://github.com/dotnet/aspnetcore/blob/0d31fa0f54474289fa6e3910f7f40d2bf8f9b5d4/src/Components/Components/src/RenderTree/Renderer.cs#L439-L493) method is responsible for dequeueig rendering tasks off the queue and processing them.

The `ProcessRenderQueue` method is invoked whenenver a render is triggered.

```csharp
private void ProcessRenderQueue()
{
    Dispatcher.AssertAccess();

    if (_isBatchInProgress)
    {
        throw new InvalidOperationException("Cannot start a batch when one is already in progress.");
    }

    _isBatchInProgress = true;
    var updateDisplayTask = Task.CompletedTask;

    try
    {
        if (_batchBuilder.ComponentRenderQueue.Count == 0)
        {
            return;
        }

        // Process render queue until empty
        while (_batchBuilder.ComponentRenderQueue.Count > 0)
        {
            var nextToRender = _batchBuilder.ComponentRenderQueue.Dequeue();
            RenderInExistingBatch(nextToRender);
        }

        var batch = _batchBuilder.ToBatch();
        updateDisplayTask = UpdateDisplayAsync(batch);

        // Fire off the execution of OnAfterRenderAsync, but don't wait for it
        // if there is async work to be done.
        _ = InvokeRenderCompletedCalls(batch.UpdatedComponents, updateDisplayTask);
    }
    catch (Exception e)
    {
        // Ensure we catch errors while running the render functions of the components.
        HandleException(e);
        return;
    }
    finally
    {
        RemoveEventHandlerIds(_batchBuilder.DisposedEventHandlerIds.ToRange(), updateDisplayTask);
        _batchBuilder.ClearStateForCurrentBatch();
        _isBatchInProgress = false;
    }

    if (_batchBuilder.ComponentRenderQueue.Count > 0)
    {
        ProcessRenderQueue();
    }
}
```

You know the drill. We'll break down the code line-by-line to understand what is going on under the hood.

**RenderInExistingBatch(nextToRender)**

We start off by dequeueing the next render entry to process and storing it in `nextToRender`. The `RenderInExistingBatch` method is responsible for taking this entry and

**_batchBuilder.ToBatch()**

A `RenderPatch` describes a set of changes that will be applied to the DOM on the next render. It captures things like:

- The set of components that were modified or rendered in a batch sequence
- 

**updateDisplayTask = UpdateDisplayAsync(batch)**

The `UpdateDisplayAsync` method is responsible for doing the actual work of rendering the updates onto the DOM. There are different implementations of the `UpdateDisplayAsync` method for the WebAssembly and Server hosting models.

The [WebAssembly implementation](https://github.com/dotnet/aspnetcore/blob/a190fd34854542266956b1af980c19afacb95feb/src/Components/WebAssembly/WebAssembly/src/Rendering/WebAssemblyRenderer.cs#L99-L107) invokes the JavaScript APIs for rendering via a JS interop call.

```csharp
DefaultWebAssemblyJSRuntime.Instance.InvokeUnmarshalled<int, **RenderBatch**, object>(
    "Blazor._internal.renderBatch",
    _webAssemblyRendererId,
    batch);
```

The [RemoteRenderer implementation](https://github.com/dotnet/aspnetcore/blob/a190fd34854542266956b1af980c19afacb95feb/src/Components/Server/src/Circuits/RemoteRenderer.cs#L147) used in the Server hosting model also implements an `UpdateDisplayAsync` method. That one is worthy of its own blog post, but suffice to say

**_ = InvokeRenderCompletedCalls(batch.UpdatedComponents, updateDisplayTask)**

`InvokRenderCompletedCalls` is responsible for invoking `NotifyRenderCompleted` for all components that have been updated in the RenderBatch. Earlier we mentioned that the `ComponentState` class implements a `NotifyRenderCompletedAsync` that calls the `OnAfterRenderAsync` methods in the Component. So, if you've ever wandered how your component's `OnAfterRender` lifecycle method gets invoked this is roughly what happens.

**HandleException(e)**

The `HandleException` logic in the `Renderer` is responsible for capturing unhandled exceptions that occur during renderer. Any exceptions in your component that aren't handled will arrive through this codepath.

**RemoveEventHandlerIds(_batchBuilder.DisposedEventHandlerIds.ToRange(), updateDisplayTask)**

**_batchBuilder.ClearStateForCurrentBatch()**

Once we've finished the acutal rendering work, we'll need to clear out the current batch from our builder. What kind of information is being cleared?

- The list of components and event handlers that were disposed
- The list components that were modified and the edits made under them