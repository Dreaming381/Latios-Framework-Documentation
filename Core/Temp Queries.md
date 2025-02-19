# Temp Queries

Temp Queries are entity queries that can be constructed and evaluated within
jobs. While not as performant as a cached `EntityQuery` from a system, Temp
Queries can come in handy when you can’t easily predict which queries you need
in advance. This is most common in scripting systems such as Unika.

## Creating a Query

You will need two things in your job in order to create and evaluate your query.
First, you need a `NativeArray<EntityArchetype>`.

```csharp
struct TempQueryExampleJob : IJob
{
    public NativeArray<EntityArchetype> archetypes;
    public EntityStorageInfoLookup      entityStorageInfoLookup;
```

You can obtain these from `EntityManager` and `SystemAPI` respectively.
`EntityStorageInfoLookup` can also be found inside a `ComponentBroker`.

```csharp
var archetypes = new NativeList<EntityArchetype>(state.WorldUpdateAllocator);
state.EntityManager.GetAllArchetypes(archetypes);

var job = new TempQueryExampleJob
{
    entityStorageInfoLookup = SystemAPI.GetEntityStorageInfoLookup(),
    archetypes              = archetypes.AsArray()
};
```

Next, in your job, you can construct a new TempQuery and pass in the two
arguments we’ve already created. The remainder of the arguments are used to
describe the query. They use the “All”, “Any”, and “None” categorization, as
well as `EntityQueryOptions`. Component enabled states are always ignored in
Temp Queries. You must specify at least 1 required component. The rest of the
arguments are optional.

Components of each category are specified via a `ComponentTypeSet`.
`ComponentTypeSet` requires quite a bit of boilerplate to populate. You can use
`TypePack<>` which implicitly casts to `ComponentTypeSet` to reduce it.

```csharp
var query = new TempQuery(archetypes, entityStorageInfoLookup, new TypePack<WorldTransform, LocalTransform, Parent>());
```

## Evaluating a Query

Once you have a `TempQuery` instance, you can easily iterate various details of
it. The most common use case is to iterate the entities that match the query.

```csharp
foreach (var entity in query.entities)
{
    // ...
```

You can also access the query’s `archetypes` and `chunks`. It is up to you to
use `ComponentLookup`, `ComponentTypeHandle`, or `ComponentBroker` to access
other data on the entities.

## Extending Evaluation

The query evaluation API is designed to be easily extendable so that you can
provide your own filtering or other logic.

A query can produce a `TempArchetypeEnumerator` which implements the
`ITempArchetypeEnumerator` interface. You can define your own implementation of
this interface and use the convenient `MatchesArchetype()` method on a
`TempQuery`. `TempArchetypeEnumerator` excludes archetypes with zero chunks for
performance.

`TempChunkEnumerator` can wrap any `ITempArchetypeEnumerator` and iterate
through the chunks in each provided archetype. This type implements the
`ITempChunkEnumerator` interface.

Next, there’s `TempMaskedChunkEnumerator` which wraps any
`ITempChunkEnumerator`. This type produces `MaskedChunk` instances, which each
contain an `ArchetypeChunk`, a `v128` bitmask, and a `bool` specifying whether
to use the bitmask. These have the same meaning as in `IJobChunk`. Note that the
provided `TempMaskedChunkEnumerator` does no filtering and always specifies the
`useEnabledMask` to `false`. To perform filtering, you would want to replace it
with an alternative implementation of the `ITempMaskedChunkEnumerator`
interface.

Lastly, `TempEntityEnumerator` can wrap any `ITempMaskedChunkEnumerator`. It
returns individual entities.

Thus, `TempQuery.entities` actually produces the following type:

```csharp
TempEntityEnumerator<TempMaskedChunkEnumerator<TempChunkEnumerator<TempArchetypeEnumerator>>>
```

## Performance Considerations

As stated above, `TempQuery` does not perform as well as Unity’s built-in cached
`EntityQuery` type. The biggest factor behind this is that `TempQuery` must scan
all archetypes to find ones that match. Future optimizations are possible. If
your use case suffers performance problems because of this, please report the
problems and specify your use case.

One universal way to mitigate this issue is to reduce the number of archetypes
your project generates. You can do this by adding and removing multiple
components at a time. Core provides extension methods to `EntityManager` and
`EntityCommandBuffer` to help with this.

Once you have a filtered list of archetypes, evaluating chunks and entities may
be slightly slower than Unity’s implementations, but not significantly slower.
