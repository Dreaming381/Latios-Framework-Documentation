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

## FluentQuery Operations

The primary methods for application code are provided as follows:

-   WithEnabled\<T\>(bool readOnly) – Adds a component to the “All” list of the
    EntityQuery
-   WithDisabled\<T\>(bool readOnly) – Adds a component to the “Disabled” list
    of the EntityQuery
-   With\<T\>(bool readOnly, bool isChunkComponent) – Adds a component to the
    “Present” list of the `EntityQuery`
    -   If any component is repeated via `WithEnabled<>()` or
        `WithDisabled<>()`, this command is effectively ignored for the
        component and the component in the other list may be altered to write
        access if required
-   Without\<T\>(bool isChunkComponent) – Adds a component to the “Absent” list
    of the EntityQuery
-   WithoutEnabled\<T\>() – Adds a component to the “None” list of the
    EntityQuery
    -   If any component is repeated via `Without<>()` or `WithDisabled<>()`,
        this command is effectively ignored for the component
-   WithAnyEnabled\<T\>(bool readOnly, bool isChunkComponent) – Adds a component
    to the “Any” list of the `EntityQuery`
    -   If any non-enableable component is repeated via `With<>()` or any
        enableable component is repeated via `WithEnabled<>()`, all
        `WithAnyEnabled<>()` invocations are ignored as the requirement has
        already been satisfied, and the component in the other list may be
        altered to write access if required
    -   If any component is repeated via `Without<>()` or `WithoutEnabled<>()`,
        this command is effectively ignored for the component
    -   If any enableable component is repeated via `WithDisabled<>()`, this
        command is effectively ignored for the component
    -   If the component is both an enableable component and a chunk component,
        its enableable state will be ignored
-   WithCollectionAspect\<T\>() – Adds the required components for the
    collection aspect
-   WithAspect\<T\>() – Adds the required components for the `IAspect`
    -   Any enableable component is added via `WithEnabled<>()` while all other
        components are added via `With<>()`
-   IncludeDisabledEntities() – Sets the `EntityQuery` to include entities with
    the `Disabled` component
-   IncludePrefabs() – Sets the `EntityQuery` to include entities with the
    `Prefab` component
-   IncludeSystemEntities() – Sets the `EntityQuery` to include entities
    associated with systems and embedded into the `SystemHandle`
-   IncludeMetaEntities() – Sets the `EntityQuery` to include meta chunks with
    the `ChunkHeader` component
-   UseWriteGroups() – Sets the `EntityQuery` to use write group filtering
-   IgnoreEnableableBits() – Sets the `EntityQuery` to ignore the implications
    of Enableable component enabled states when iterating, as if the value of
    the states were always set to whatever would make the entity match the
    query’s desires
-   WithDelegate() – Allows specifying a delegate to manipulate the Fluent Query
    while preserving the fluent-style code flow
-   Build() – Finishes specifying components and creates a real `EntityQuery`

## Extending FluentQuery

`FluentQuery` is designed to be extendible, and performs extra logic to resolve
trivial conflicts such as duplicate components in the same category while
promoting read-only declarations to read-write declarations when required. This
allows packages to specify parts of queries they require without creating
conflicts with the components users specify.

To extend Fluent Query, simply create an extension method which returns the
result of the last element in the chain continued inside the extension method.

One use case is to quickly add a group of components commonly used together
using a single method.

Example:

```csharp
public static FluentQuery WithExposedBone(this FluentQuery fluent, bool readOnly)
{
    return fluent.WithAspect<TransformAspect>().With<BoneIndex>(readOnly).With<BoneOwningSkeletonReference>(readOnly);
}
```

Another use case is for a library providing an API that requires an
`EntityQuery` with a minimum set of read-only components, but would be satisfied
if the user gave an `EntityQuery` with write access instead.

When multiple extension methods are used, it can be very easy to create
read-write vs read-only conflicts. Likewise, it is easy for a user to try and
exclude a component an extension method calls `WithAnyEnabled()` on. Fluent
Queries have special rules to automatically resolve many of these types of
conflicts. However, there are still some cases that are impossible to resolve.

## Fluent Queries vs Unity Query Builders

Fluent Queries were created before the modern implementation of
`EntityQueryBuilder` and `SystemAPI.QueryBuilder`. The vocabulary between APIs
are a bit different, and some APIs don’t have equivalents yet.

| EntityQueryBuilder                       | Fluent                      |
|------------------------------------------|-----------------------------|
| WithAll                                  | WithEnabled(true)           |
| WithAllRW                                | WithAll()                   |
| WithAllChunkComponent                    | With(true, true)            |
| WithAllChunkComponentRW                  | With(false, true)           |
| WithAll(listOfTypes)                     | X                           |
| WithAny                                  | WithAnyEnabled(true)        |
| WithAnyRW                                | WithAnyEnabled(false)       |
| WithAnyChunkComponent                    | WithAnyEnabled(true, true)  |
| WithAnyChunkComponentRW                  | WithAnyEnabled(false, true) |
| WithAny(listofTypes)                     | X                           |
| WithNone                                 | WithoutEnabled              |
| WithNoneChunkComponent                   | Without(true)               |
| WithNone(listOfTypes)                    | X                           |
| WithDisabled                             | WithDisabled(true)          |
| WithDisabledRW                           | WithDisabled(false)         |
| WithDisabled(listOfTypes)                | X                           |
| WithAbsent                               | Without                     |
| WithAbsentChunkComponent                 | Without(true)               |
| WithAbsent(listOfTypes)                  | X                           |
| WithPresent                              | With(true)                  |
| WithPresentRW                            | With(false)                 |
| WithPresentChunkComponent                | With(true, true)            |
| WithPresentChunkComponentRW              | With(false, true)           |
| WithPresent(listOfTypes)                 | X                           |
| WithAspect                               | WithAspect                  |
| X                                        | WithCollectionAspect        |
| X                                        | WithDelegate                |
| WithOptions(IncludePrefab)               | IncludePrefab               |
| WithOptions(IncludedisabledEntities)     | IncludeDisabledEntities     |
| WithOptions(FilterWriteGroup)            | UseWriteGroups              |
| WithOptions(IgnoreComponentEnabledState) | IgnoreEnableableBits        |
| WithOptions(IncludeSystems)              | IncludeSystemEntities       |
| WithOptions(IncludeMetaChunks)           | IncludeMetaEntities         |
| AddAdditionalQuery                       | X                           |
