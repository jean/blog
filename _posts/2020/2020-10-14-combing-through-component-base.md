---
layout: posts
title: Combing through ComponentBase
tags: [blazor]
---

This is the first blog post in a mini-series on the internals of rendering in Blazor. As a preface, this blog post assumes that you're familiar with [Blazor](https://blazor.net/).

Blazor, like many modern web frameworks, provides a component-model that allows users to colocate UI elements that represent shared business logic into single units, known as components. React, Vue, Angular, and Svetle all have similar concepts in their respective implementations. The framework's component model ties into other key concepts in the framework associated with event handling, rendering, and data management. So, as it turns out, taking a look at framework's component implementation is a great way to get a sense of the framework overall.

### In the beginning, there was ComponentBase

The main character in Blazor's component model is the `ComponentBase` class ([ref](https://github.com/dotnet/aspnetcore/blob/795584e177852a2cb95d4e7b0df2ae7d94fac157/src/Components/Components/src/ComponentBase.cs)). The first thing to examine is the class declaration for `ComponentBase` itself:

```csharp
public abstract class ComponentBase : IComponent, IHandleEvent, IHandleAfterRender
```

This is only one line of code but we can actually unpack quite a bit from it.

For starters, `ComponentBase` is an `abstract` (aka base) class. You cannot instantiate an instance of the `ComponentBase` class itself, but most Blazor components derive from the `ComponentBase` class itself.

Most? Not all? Correct! Not all components have to implement `ComponentBase`. In fact, the `Router` component that we looked at in last week's blog post doesn't derive from `ComponentBase`. It implements the much narrower `IComponent` interface ([ref](https://github.com/dotnet/aspnetcore/blob/795584e177852a2cb95d4e7b0df2ae7d94fac157/src/Components/Components/src/IComponent.cs)).

```csharp
public interface IComponent {
    void Attach(RenderHandle renderHandle);
    Task SetParametersAsync(ParameterView parameters);
}
```

Pretty bare bones, huh? Notably, the `IComponent` interface does not require that the component implement Blazor's component lifecycle methods. We'll cover these lifecycle methods later in the chapter, but suffice to say that 

### The Lifecycle Methods

Whenever I think about lifecycle methods in components (whether Blazor or React), I always think back to those "Life Cycle of a Cell" diagrams from high school. The concepts are actually fairly similar. A component has particular stages of development, from when it is first rendered on the page to when it is discarded. Components can also react to changes in their "environment", whether it is events being triggered or parameters changing.

#### SetParametersAsync

After a component is initialized, we need to establish what parameters have been provided to the component. This needs to happen before any rendering occurs because what is rendered is often dependent on the parameters provided to the component.

##### OnInitialized/OnInitializedAsync

For each lifecycle method, there typically exists an asynchronous and synchronous varaint.

##### OnParametersSet/OnParmaetersSetAsync

##### OnAfterRender/OnAfterRenderAsync

### A RenderTree for you and me





