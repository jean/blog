```
---
title: A deep dive into endpoint filters in ASP.NET Core 7
---
```

One of the features that I worked on as part of the ASP.NET Core 7 release was support for endpoint filters in minimal APIs. I recently shared some of the internals of the implementation at a recent Ignite presentation, but I figured I would share the insights in a blog as well. If you're more of a visual or audio learner, you might find the video presentation helpful.

Heads up: this is an advanced blog post and assumes that you are familiar with ASP.NET and minimal APIs.

Let's get started with a quick overview of what endpoint filters are. Endpoint filters are similar to MVC's action filters in the sense that they allow you to introspect and modify parameters and also intercept responses issued by a request. However, they are designed to work on the endpoints that are the core primitive of ASP.NET's routing logic, as opposed to the controller-specific action filters.

With that in mind, let's take a look at an example of a route endpoint with an endpoint filter.

```csharp
app.MapGet("/hello/{name}", (string name) => $"Hello {name}!")
    .AddEndpointFilterFactory((context, next) =>
    {
        var parameters = context.MethodInfo.GetParameters();
        // Only operate handlers with a single argument
        if (parameters.Length == 1 &&
            parameters[0] is ParameterInfo parameter &&
            parameter.ParameterType == typeof(string))
        {
            return invocationContext =>
            {
                // Map the first string argument we
                // receive to an upper-case string
                var modifiedArgument = invocationContext
                    .GetArgument<string>(0)
                    .ToUpperInvariant();
                invocationContext.Arguments[0] = modifiedArgument;
                return next(invocationContext);
            };
        }

        return invocationContext => next(invocationContext);
    });
```

There's a couple of things that we will want to observe here:

- We're invoking a `AddEndpointFilterFactory` method that appears to take a context parameter, a next parameter, and returns a delegate.
- There's another context referenced in that delegate, referred to as the `invocationContext`.
- The context in the factory appears to give us access to a `MethodInfo` that we can introspect to analyze the signature of the handler. In this case, we're able to examine that it processes a single parameter that happens to be a string.

There's a handful of key concepts that are introduced here that we will dig into further across the rest of the blog post. For now, I want to dive a little bite more into the `AddEndpointFilterFactory` method as a gateway to explore the rest of the implementation of filters here. The `AddEndpointFilterFactory` is implemented [exactly as follows](https://github.com/dotnet/aspnetcore/blob/3c36e1643045d8c7baebece79ccd901321dfe5f2/src/Http/Routing/src/Builder/EndpointFilterExtensions.cs#L110-L118):

```csharp
    public static TBuilder AddEndpointFilterFactory<TBuilder>(this TBuilder builder, Func<EndpointFilterFactoryContext, EndpointFilterDelegate, EndpointFilterDelegate> filterFactory)
        where TBuilder : IEndpointConventionBuilder
    {
        builder.Add(endpointBuilder =>
        {
            endpointBuilder.FilterFactories.Add(filterFactory);
        });

        return builder;
    }
```

The signature of this method validates the observations that we saw earlier about the inputs/returns of filter factories. That `context` object we interacted with earlier is actaully an `EndpointFilterFactoyContext`. Both the `next` and the delegate returned from the filter factory, are `EndpointFilterDelegate`s which have the following signature:

```csharp
public delegate ValueTask<object?> EndpointFilterDelegate(EndpointFilterInvocationContext context);
```

Here, we see our second context type: the `EndpointFilterInvocationContext`. We'll also note that `EndpointFilterDelegate`s must return a `ValueTask<object?>`.

The `AddEndpointFilterFactory` exposes us to another key element, the fact that the `FilterFactories` are stored in the endpoint builder. More specifically, the `FilterFactories` sequence is a required property of the base `EndpointBuilder`.

