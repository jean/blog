---
layout: posts
title: Rolling in render trees
tags: [blazor]
---

This is the second blog post in a mini-series on the internals of rendering and components in Blazor. By internals, I mean that we’ll be doing one of my favorite things: reading code and figuring out what it does. As a preface, this blog post assumes that you’re familiar with [Blazor](https://blazor.net).

If you haven't read the first blog post already, I'd recommend [taking a look at it](./combing-through-component-base) before reading the rest of this post. We'll be picking up right where we left off.

### Building render trees

In the last blog post, we discussed the `ComponentBase` class in Blazor. Each component has a `StateHasChanged` method that is invoked when the component needs to be rendered. We'll use this method as a transition from our exploration of `ComponentBase` to our dive into render trees.

```csharp
protected void StateHasChanged()
{
    if (_hasPendingQueuedRender)
    {
        return;
    }

    if (_hasNeverRendered || ShouldRender())
    {
        _hasPendingQueuedRender = true;

        try
        {
            _renderHandle.Render(_renderFragment);
        }
        catch
        {
            _hasPendingQueuedRender = false;
            throw;
        }
    }
}
```

The relevant parts of this method are the invocation of `_renderHandle.Render(_renderFragment)` in the try-catch block. We'll start our exploration by looking at render handles.

#### Getting a handle on render handles

A render handle is a construct that allows a component to interact with its handler, so it _definitely_ is a great transition from talking about components to talking about rendering. At it's core, the `RenderHandle` maintains a reference to two key properties: the renderer and an integer representing the ID of the component the render handle is associated with. The `Render` method invoked above calls into the `AddToRenderQueue` method on the `Renderer` to facilitate rendering.

The `RenderHandle` class is onl 60-odd lines of code (and fifteen or so if you ignore whitespace nad newlines.) You can check out the code for it [here](https://github.com/dotnet/aspnetcore/blob/0d31fa0f54474289fa6e3910f7f40d2bf8f9b5d4/src/Components/Components/src/RenderHandle.cs).

#### Biting into the Renderer

Unlike the `RenderHandle`, the `Renderer` implementation has a fair amount of logic. We'll break into the `Renderer` by taking a look at the [`AddToRenderQueue`](https://github.com/dotnet/aspnetcore/blob/0d31fa0f54474289fa6e3910f7f40d2bf8f9b5d4/src/Components/Components/src/RenderTree/Renderer.cs#L377-L396) method that's invoked above.

```csharp
internal void AddToRenderQueue(int componentId, RenderFragment renderFragment)
{
    Dispatcher.AssertAccess();

    var componentState = GetOptionalComponentState(componentId);
    if (componentState == null)
    {
        return;
    }

    _batchBuilder.ComponentRenderQueue.Enqueue(new RenderQueueEntry(componentState, renderFragment));

    if (!_isBatchInProgress)
    {
        ProcessPendingRender();
    }
}
```

Let's break down what is going on here line-by-line.

##### Dispatcher.AssertAccess()

The `Dispatcher`is responsible for 

##### var componentState = GetOptionalComponentState(componentId)

A `ComponentState` object is used to the track the current rendering state of the component. It's responsible for a few things including:

- Subscribing to changes in [cascading parameters](https://docs.microsoft.com/en-us/aspnet/core/blazor/components/cascading-values-and-parameters) and notifying the component to update its parameters if they've been changed
- Notifying the component when [rendering has been completed](https://github.com/dotnet/aspnetcore/blob/6a82a84b10a8073f262ea53d7f8e01fa6cbfca08/src/Components/Components/src/Rendering/ComponentState.cs#L120-L136) to trigger any `OnAfterRender` lifecycle methods
- Maintaining a reference to the parent component's `ComponentState` which is useful for tracking cascading parameters

You'll notice in the logic above that we don't invoke any rendering if we don't have a `ComponentState` object registered for the current component. The `Renderer` maintains a dictionary of { componentId: ComponentState } and exposes two variants for accessing a `ComponentState` by ID.

- `GetOptionalComponentState` returns the `ComponentState` for a given component ID or returns null. This is typically used in scenarios where we want to respond to a component that has already been disposed. Components that have been disposed can't re-render.
- `GetRequiredComponentState` returns the `ComponentState` for a given component ID and throws an exception if none is found. This more aggressive approach is used in code paths where we want to gurantee that a `ComponentState` exists before using it, such as before disposing of component state.

In the case above, we avoid queueing a new render operation if the component has already been disposed.

##### _batchBuilder

The `_batchBuilder` property is an instance of a [`RenderBatchBuilder`](https://github.com/dotnet/aspnetcore/blob/master/src/Components/Components/src/Rendering/RenderBatchBuilder.cs) associated with the current renderer.

Render batches are 

##### .ComponentRenderQueue

The `ComponentRendereQueue` maintains the set of the rendering events that still need to be completed by the renderer.

##### .Enqueue(new RenderQueueEntry(componentState, renderFragment))

Before performing a render, we first add it to the render queue. If no renders are currently in progress, we process it immediately (more on this in the next section). If there is a render in progress 

##### ProcessPendingRender()

The `ProcessPendingRender` method is invoked if we can process remaining renders off the render queue. If the renderer hasn't been disposed, [the method](https://github.com/dotnet/aspnetcore/blob/0d31fa0f54474289fa6e3910f7f40d2bf8f9b5d4/src/Components/Components/src/RenderTree/Renderer.cs#L429-L437) will invoke `ProcessRenderQueue`.

### ProcessRenderQueue

In previous sections, we discussed the concept of a render queue in which upcoming render batches are enqueued and dequeued throughout the lifespan of the renderer. The [`ProcessRenderQueue`](https://github.com/dotnet/aspnetcore/blob/0d31fa0f54474289fa6e3910f7f40d2bf8f9b5d4/src/Components/Components/src/RenderTree/Renderer.cs#L439-L493) method is responsible for dequeueig rendering tasks off the queue and processing them.

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

        while (_batchBuilder.ComponentRenderQueue.Count > 0)
        {
            var nextToRender = _batchBuilder.ComponentRenderQueue.Dequeue();
            RenderInExistingBatch(nextToRender);
        }

        var batch = _batchBuilder.ToBatch();
        updateDisplayTask = UpdateDisplayAsync(batch);

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

Let's break down this method line-by-line like we did with `AddToRenderQueue` above. Note: we'll be skipping through the more obvious lines in this exploration.

##### RenderInExistingBatch(nextToRender)

##### _batchBuilder.ToBatch()

##### updateDisplayTask = UpdateDisplayAsync(batch)

##### _ = InvokeRenderCompletedCalls(batch.UpdatedComponents, updateDisplayTask)

##### HandleException(e)

#####  RemoveEventHandlerIds(_batchBuilder.DisposedEventHandlerIds.ToRange(), updateDisplayTask)

##### _batchBuilder.ClearStateForCurrentBatch()

## Conclusion

