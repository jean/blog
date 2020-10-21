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

Unlike the `RenderHandle`, the `Renderer` implementation has a fair amount of logic. We'll break into the `Renderer` by taking a look at the `AddToRenderQueue` method that's invoked above.

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

##### Dispatcher.AssertAccess()

The `Dispatcher`is responsible for 

##### var componentState = GetOptionalComponentState(componentId);

A `ComponentState` object is used to the track the current rendering state of the component. It's responsible for tracking 

##### _batchBuilder.ComponentRenderQueue.Enqueue(new RenderQueueEntry(componentState, renderFragment));

##### ProcessPendingRender();

