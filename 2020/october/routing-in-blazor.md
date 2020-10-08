---
layout: post
title: Under the hood with routing in Blazor
---

A while back, I posted [a response to a user issue](https://github.com/dotnet/aspnetcore/issues/21668#issuecomment-626909464) on the ASP.NET Core repo explaining the inner workings of routing in Blazor. The response was pretty good, but I wanted to take the oppurtunity to flesh it out a little bit more in a blog post.

Ready?

Here goes!

Routing is a pretty fundamental part of a lot of SPA (single-page application) frameworks. React has [React Router](https://reach.tech/router/). Vue has [Vue router](https://router.vuejs.org/). Angular has [built-in support](https://angular.io/guide/router) for routing. Routers are an important aspect of SPAs, because unlike traditional websites, SPAs don't have the luxury of going to the server to fetch a new pag when the user navigates to a route.

We can break down the task of routing in a single-page application into a few key portions:

- A way to register routes and the pages that are associated with them
- A way to map a URL that a user visits to the page registered with it

We'll start our exploration of routing by taking a look at the first part: how do we actually define routes and associate them with pages (or more precisely, components)?

### The TemplateParser
The magic starts in the [TemplateParser](https://github.com/captainsafia/aspnetcore/blob/403bdaa1a6144d791c8ca38c112c39c7e51def17/src/Components/Components/src/Routing/TemplateParser.cs) class which interprets what the different components of a string route template are. It contains a single static method definition that processes a string representation of a template.

```csharp
internal static RouteTemplate ParseTemplate(string template)
```

In particular, it takes a route like `/students/{StudentId:int}` and splits it by `/` to get each path-separated segment. Then parses each path segment to determine whether the segment is a static value (e.g. "students") or contains a parameter (e.g. "StudentId:int"). From this parsing, it produces a [RouteTemplate](https://github.com/captainsafia/aspnetcore/blob/2535bb127d4d99e7d008991aa74d8a05c946c0a5/src/Components/Components/src/Routing/RouteTemplate.cs) object.

### The RouteTemplate
Each RouteTemplate object contains a set of [TemplateSegment](https://github.com/captainsafia/aspnetcore/blob/403bdaa1a6144d791c8ca38c112c39c7e51def17/src/Components/Components/src/Routing/TemplateSegment.cs) objects that match to each `/`-seperated segment of the route. This `TemplateSegment` provides some information about whether or not the segment is related to a `Parameter` and exposes a `Match` function that takes in an input segment and checks to see if it matches the TemplateSegment. Note that this `Match` function also extracts parameter matches.

The `Match` function implemented in the `TemplateSegment` is invoked in the [RouteEntry](https://github.com/captainsafia/aspnetcore/blob/2535bb127d4d99e7d008991aa74d8a05c946c0a5/src/Components/Components/src/Routing/RouteEntry.cs#L28) class. The TL;DR of this class is that it processes a given path, like `/students/123` and finds the route that has the most number of matching templates segments. It also stores all the parameters used in the route in the `RouteContext` object.

### The Router
So at this point, the parameters passed in the URL are loaded into the parameter object in the route context. It's used in the [Route](https://github.com/captainsafia/aspnetcore/blob/2535bb127d4d99e7d008991aa74d8a05c946c0a5/src/Components/Components/src/Routing/Router.cs#L115) component which renders the route with a `RouteData` object that contains the parameter using the `Found` render fragment.

```
<Router AppAssembly=typeof(StandaloneApp.Program).Assembly>
    <Found Context="routeData">
        <RouteView RouteData="@routeData" DefaultLayout="@typeof(MainLayout)" />
    </Found>
    <NotFound>
        <LayoutView Layout="@typeof(MainLayout)">
            <h2>Not found</h2>
            Sorry, there's nothing at this address.
        </LayoutView>
    </NotFound>
</Router>
```

The example above shows a standard `App` component setup with the `Found` render fragment. Here, you can see that this is rendering a [RouteView](https://github.com/captainsafia/aspnetcore/blob/2535bb127d4d99e7d008991aa74d8a05c946c0a5/src/Components/Components/src/RouteView.cs) component.

The `RouteView` can [render the page with the parameters mapped in as attributes](https://github.com/captainsafia/aspnetcore/blob/2535bb127d4d99e7d008991aa74d8a05c946c0a5/src/Components/Components/src/RouteView.cs#L78). The [RenderTreeBuilder](https://github.com/captainsafia/aspnetcore/blob/2535bb127d4d99e7d008991aa74d8a05c946c0a5/src/Components/Components/src/Rendering/RenderTreeBuilder.cs#L186) provides an `AddAttribute` API for setting these properties.

Once the attributes are set, the [ParameterView](https://github.com/captainsafia/aspnetcore/blob/2535bb127d4d99e7d008991aa74d8a05c946c0a5/src/Components/Components/src/ParameterView.cs) component provides functionality for extracting the parameters from the render tree. In particular, it contains a [SetParameterProperties](https://github.com/captainsafia/aspnetcore/blob/2535bb127d4d99e7d008991aa74d8a05c946c0a5/src/Components/Components/src/ParameterView.cs#L234) method that updates each of the parameters on the component to the values mapped by the ParameterView (which come from the RenderTree fragment which comes from the RouteView which is invoked in by the Route component).

The [ComponentProperties](https://github.com/captainsafia/aspnetcore/blob/2535bb127d4d99e7d008991aa74d8a05c946c0a5/src/Components/Components/src/Reflection/ComponentProperties.cs) processes the [attributes in the component](https://github.com/captainsafia/aspnetcore/blob/2535bb127d4d99e7d008991aa74d8a05c946c0a5/src/Components/Components/src/Reflection/ComponentProperties.cs#L226) and [updates their values accordingly](https://github.com/captainsafia/aspnetcore/blob/2535bb127d4d99e7d008991aa74d8a05c946c0a5/src/Components/Components/src/Reflection/ComponentProperties.cs#L38).

TL;DR: The process of mapping a URL parameter to a component properties is supported by shared data structures like the RouteContext and various classes/methods that build the mapping from: string URL -> parsed route -> parameter object -> parameters injected into view -> parameters mapped to properties.
