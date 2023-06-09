# Fluent Queries

`FluentQuery` is a struct with a fluent builder API for constructing
`EntityQuery` instances. It aims to accommodate use cases where partial queries
across different packages need to be aggregated together.

## Creating EntityQueries using FluentQuery

You can create a base `FluentQuery` instance using the `Fluent` property on
`SubSystem` and `SuperSystem`, or you can use the `EntityManager` and
`SystemState` extension method `Fluent()`.

To convert a `FluentQuery` into an `EntityQuery`, call `Build()`.

While unnecessary for typical use cases, you can store a `FluentQuery` instance
in a local variable or pass it to functions. However, the base `FluentQuery`
instance allocates native containers on construction and those containers are
only disposed when `Build()` is called.

## Basic FluentQuery Operations

The primary methods for application code are provided as follows:

-   WithAll\<T\>(bool readOnly) – Adds a component to the “All” list of the
    `EntityQuery`
-   WithAny\<T\>(bool readOnly) – Adds a component to the “Any” list of the
    `EntityQuery`
    -   If any component is repeated in the “All” list, the “Any” list is
        dropped entirely
-   Without\<T\>() – Excludes the component from the resulting `EntityQuery`
-   IncludeDisabled() – Sets the `EntityQuery` to use disabled entities
-   IncludePrefabs() – Sets the `EntityQuery` to use prefab entities
-   UseWriteGroups() – Sets the `EntityQuery` to use write group filtering

## Extending FluentQuery

Unlike `Entities.ForEach`, `FluentQuery` is extendible. To extend it, simply
create an extension method which returns the result of the last element in the
chain continued inside the extension method.

One use case is to quickly add a group of components commonly used together
using a single method.

Example:

```csharp
public static FluentQuery WithCommonTransforms(this FluentQuery fluent, bool readOnly)
{
    return fluent.WithAll<Translation>(readOnly).WithAll<Rotation>(readOnly).WithAll<LocalToWorld>(readOnly);
}
```

Another use case is for a library providing an API that requires an
`EntityQuery` with a minimum set of ReadOnly components, but would be satisfied
if the user gave an `EntityQuery` with ReadWrite access instead.

When multiple extension methods are used, it can be very easy to create
ReadWrite vs ReadOnly conflicts. Likewise, it is easy for a user to try and
exclude a component an extension method calls `WithAny()` on. For this reason,
some advanced API exists so that extension methods can specify their minimum
requirements, allowing other extension methods or user code to override the
requests as long as the minimum requirements are met.

These advanced methods are as follows:

-   WithAllWeak\<T\>() – Adds a component to the “All” list as ReadOnly unless
    something else in the FluentQuery adds the component as ReadWrite
-   WithAnyWeak\<T\>() – Same as `WithAllWeak<T>()` except applied to the “Any”
    list
    -   If any component is repeated in the “All” list, the “Any” list is
        dropped entirely
-   WithAnyNotExcluded\<T\>(bool readOnly) – Adds the component to the “Any”
    list unless it is excluded using `Without<T>()`
-   WithAnyNotExcludedWeak\<T\>() – Applies both effects of `WithAnyWeak<T>()`
    and `WithAnyNotExcluded<T>(false)`

## Fluent Queries vs Unity Query Builders

Fluent Queries were created before the modern implementation of
`EntityQueryBuilder` and `SystemAPI.QueryBuilder`. At the time of writing,
Entity Queries don’t have full combination coverage within the API. As such,
work has not been done yet to rectify differences in the APIs. The following
table outlines the differences and discrepancies. The “Any” flavor APIs are
equivalent to “All” flavors for the purpose of this comparison unless explicitly
tabulated.

| EntityQueryBuilder                       | Fluent                    |
|------------------------------------------|---------------------------|
| WithAll                                  | WithAll(true)             |
| WithAllRW                                | WithAll()                 |
| WithAllChunkComponent                    | WithAll(true, true)       |
| WithAllChunkComponentRW                  | WithAll(false, true)      |
| WithAll(listOfTypes)                     | X                         |
| X                                        | WithAllWeak               |
| X                                        | WithAny/NotExcluded/Weak/ |
| WithNone                                 | Without                   |
| WithNoneChunkComponent                   | Without(true)             |
| WithNone(listOfTypes)                    | X                         |
| WithDisabled/RW                          | X                         |
| WithAbsent/ChunkComponent                | X                         |
| WithAspect                               | X                         |
| X                                        | WithDelegate              |
| WithOptions(IncludePrefab)               | IncludePrefab             |
| WithOptions(IncludedisabledEntities)     | IncludeDisabledEntities   |
| WithOptions(FilterWriteGroup)            | UseWriteGroups            |
| WithOptions(IgnoreComponentEnabledState) | IgnoreEnableableBits      |
| WithOptions(IncludeSystems)              | X                         |
| WithOptions(IncludeMetaChunks)           | X                         |
| AddAdditionalQuery                       | X                         |
