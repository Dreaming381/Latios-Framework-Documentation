# ILatiosApi, IInjectable, and IJobEach

Unity’s ECS requires heavy caching of types in systems, resulting in a lot of
boilerplate. Unity tries to solve this with `SystemAPI` and `IJobEntity` backed
by source generators. The solutions work okay for the common use cases, but fall
apart when trying to extend them. Plus they don’t even eliminate all
boilerplate.

The Latios Framework ships its own boilerplate-reducing APIs and source
generators to solve these problems. These new APIs can sit alongside Unity’s
without conflict, even in the same system. Note that only unmanaged `ISystem`
systems are supported.

## ILatiosApi

`ILatiosApi` is the interface that your `ISystem` implements. You don’t have to
implement any of its methods. Instead, you need to start `OnCreate()` like this:

```csharp
[BurstCompile]
public void OnCreate(ref SystemState state)
{
    var api = this.OnCreateForLatios(ref state);
}
```

*Note:* `[BurstCompile]` *and* `var api =` *can be dropped if not needed.*

`api` provides access to source-generated cached values. It can also be passed
into some methods to inject things. Outside of `OnCreate()`, you can fetch it
via

```csharp
var api = this.GetApi(ref state); 
```

All usages of `api` must be within your `ISystem` (in any partial
implementation). If you try to make external static methods and pass it around,
or fetch it from another system, you will most likely get runtime errors unless
you happen to use something that system generated a cache for anyways.

`api` provides the following properties for convenience:

-   `latiosWorld`
-   `syncPoint`
-   `tickedSyncPoint`
-   `worldBlackboardEntity`
-   `sceneBlackboardEntity`
-   `deltaTime`
-   `elapsedTime`
-   `time`
-   `lastSystemVersion`

All of those are always available for any valid `api`. Additionally, `api`
provides various `Get*()` methods for accessing common cached handles. These
include:

-   `GetComponentHandle<T>(bool readOnly)`
-   `GetBufferHandle<T>(bool readOnly)`
-   `GetSharedComponentHandle<T>()`
-   `GetComponentLookup<T>(bool readOnly)`
-   `GetBufferLookup<T>(bool readOnly)`
-   `GetEntityHandle()`
-   `GetEntityStorageInfoLookup()`

It also contains two special methods:

-   `Get<T>()`
-   `Get<T>(bool b)`

In these two methods, `T` must be of type `ILatiosApiGettable` or
`ILatiosApiGettableBool`. These are interfaces you can implement yourself by
implementing `CreateForApi()` and `UpdateForApi()` which pass in the
`SystemState`. `CreateForApi()` is invoked on a default-initialized instance.
`ILatiosApiGettableBool` will also pass in the `bool` value into
`CreateForApi()` that was (or will be) passed into the `Get<T>(bool b)` method.
You typically do this to implement a custom aggregate handle. Here’s an abridged
example from Psyshock:

```csharp
public struct BuildCollisionLayerTypeHandles : ILatiosApiGettable
{
    [ReadOnly] public ComponentTypeHandle<Collider>           collider;
    [ReadOnly] public WorldTransformReadOnlyAspect.TypeHandle worldTransform;
    [ReadOnly] public EntityTypeHandle                        entity;

    public BuildCollisionLayerTypeHandles(ref SystemState system)
    {
        collider       = system.GetComponentTypeHandle<Collider>(true);
        worldTransform = new WorldTransformReadOnlyAspect.TypeHandle(ref system);
        entity         = system.GetEntityTypeHandle();
    }

    public void Update(ref SystemState system)
    {
        collider.Update(ref system);
        worldTransform.Update(ref system);
        entity.Update(ref system);
    }

    void ILatiosApiGettable.CreateForApi(ref SystemState state) => this = new BuildCollisionLayerTypeHandles(ref state);

    void ILatiosApiGettable.UpdateForApi(ref SystemState state) => Update(ref state);
}
```

## IInjectable

Already, we’ve seen a feature beyond `SystemAPI` in being able to define custom
aggregate cached types. But why fetch handles at all when we could just have the
job fetch its own handles? That would save a lot of job setup code.

Guess what? `IInjectable` does this.

To use it, simply define any struct (yes, any struct, not just a job) as partial
and implementing `IInjectable`. Do not implement any of `IInjectable`’s methods
yourself.

Then, add the `[Inject]` attribute on any type that is supported by
`api.Get*()`. The fields can be private. If you use `[ReadOnly]` on an
`ILatiosApiGettableBool` type, `true` will be passed into the type’s creation.
Here’s an example of `IInjectable`’s job definition (excluding methods):

```csharp
[BurstCompile]
partial struct PropoagateNewChildrenAfterExistingRestoreJob : IJob, IInjectable
{
    [ReadOnly] public NativeStream.Reader stream;
    [ReadOnly, Inject] EntityTypeHandle   entityHandle;
    [Inject] TickedTransformAspectLookup  transformAspectLookup;
```

To populate the handles, simply call `.Inject(api)` on the struct. The method
returns the populated struct so you can chain the scheduling like this:

```csharp
jh = new PropoagateNewChildrenAfterExistingRestoreJob
{
    stream = stream.AsReader(),
}.Inject(api).Schedule(jh);
```

For jobs that use a lot of handles and lookups, this can drastically improve the
readability of your scheduling code. For very large jobs, `.InjectByRef(api)` is
also an option.

## IJobEach

`IJobEach` is an alternative to `IJobEntity`. It exists to solve real pain
points users have encountered with `IJobEntity` especially when paired with the
Latios Framework. `IJobEach` preserves much of the same workflow as
`IJobEntity`, but with several crucial new features.

At its core, you declare an IJobEach the same way you would declare an
IJobEntity:

```csharp
[BurstCompile]
partial struct Job : IJobEach
{
    public void Execute(ref FadeProperty fade, in TimeToLive timeToLive, in TimeToLiveFadeStart timeToLiveFadeStart)
    {
        fade.fade  = math.saturate(timeToLive.timeToLive / timeToLiveFadeStart.fadeTimeWindow);
        fade.fade *= fade.fade;
    }
}
```

That’s three `IComponentData` types passed in, one by `ref`, and the other two
by `in`, just like `IJobEntity`.

Scheduling is equally familiar:

```csharp
new Job().ScheduleParallel(api);
```

Here, the `EntityQuery` is implicitly defined by the job, and `state.Dependency`
is read from and assigned to automatically. Again, it is the same as
`IJobEntity`, except this time `api` is passed into the scheduler.

You can of course pass in a custom `EntityQuery` or manually control
`JobHandle`s just like `IJobEntity` too. `api` is always the first parameter
when scheduling.

### Supported Execute Parameters

`IJobEach` supports many of the same types of parameters as `IJobEntity`.
However, there are a few differences. `IJobEach` supports:

-   `Entity`
-   `IComponentData` (including `in` and `ref` modifiers)
-   `DynamicBuffer<IBufferElementData>` (including in and ref modifiers)
-   `RefRW<T>` and `RefRO<T>`
-   `EnabledRefRW<T>` and `EnabledRefRO<T>`
-   `EnableableRefRW<T>` and `EnableableRefRO<T>`
-   `IParameter`
-   `in IJobEach.JobContext`

Those last three might be foreign to you. That’s because they are all defined by
the framework.

`EnableableRefRW<T>` combines `RefRW<T>` and `EnabledRefRW<T>` into a single
type for convenience.

`IParameter` is a struct implementing the `IParameter` interface, and is the
primary hook that makes `IJobEach` extensible. There are many built-in
`IParameters` throughout the framework. You can also write your own (more on
that later).

`JobContext` provides a bunch of metadata about the job. It contains the
following properties:

-   `chunk` – The currently processing chunk
-   `indexInChunk` – The index of the currently processing entity in the chunk
-   `chunkIndexInQuery` – The index of the chunk within the `EntityQuery`
-   `chunkEnabledMask` – The `IEnableableComponent` filtering mask applied to
    the chunk
-   `useEnabledMask` – Whether or not `chunkEnabledMask` is valid
-   `mode` – The scheduling mode used, such as `Run()` or `ScheduleParallel()`
-   `lastSystemVersion` – The `SystemState`’s `LastSystemVersion` when the job
    was dispatched
-   `entityIndexInQuery` – The index of the entity in the compacted
    `EntityQuery`
-   `hasEntityIndexInQuery` – Whether or not `entityIndexInQuery` is valid

`indexInChunk`, `chunkIndexInQuery`, and `entityIndexInQuery` replace the three
`int` parameter types of `IJobEntity`. However, `entityIndexInQuery` is not
always valid when the job is scheduled using `ScheduleParallel()` due to the
additional overhead it creates. To inform the job that it is needed, add
`[RequireEntityIndexInQuery]` to the job. In some cases, an `IParameter` may
force it to be valid even without that attribute.

### Execute Parameter Attributes

`IJobEach` recognizes two attributes that can be used on parameters directly.

`[WithEnabled]` supports on any type of parameter that is an
`IEnableableComponent`. Unlike `IJobEntity`, parameters are **not** required to
be enabled by default. This attribute requires the component be enabled for the
entity to be processed. This attribute may be ignored if a fully custom
`EntityQuery` is provided to the job.

`[NotRequired]` works on `RefRW`, `EnabledRefRW`, `EnableableRefRO`, the
read-only counterparts, and `DynamicBuffer`. The attribute specifies that the
component is not required to be part of the entity’s archetype. If it is not,
then `IsValid` or `IsCreated` will be `false`.

**Warning**: `RefRW` and `RefRO` on a zero-sized component always return `false`
for `IsValid`. Use the `IParameter` `Has<T>` instead.

`IParameters` can have their own special interactions with attributes. For
built-in parameters, refer to the `IParameter`’s documentation to see if it
supports any attributes that alter its behavior. Most `IParameters` don’t.

### Job Attributes

Similar to `IJobEntity`, `IJobEach` supports various attributes for refining the
default `EntityQuery`. However, unlike `IJobEntity`, `IJobEach` uses
`FluentQuery` rules. And consequently, it has its own set of parameters:

-   `[WithEnabled]`
-   `[With]`
-   `[Unity.Entities.WithDisabled]`
-   `[WithAnyEnabled]`
-   `[Without]`
-   `[WithoutEnabled]`
-   `[IngoreEnableableBits]`
-   `[IncludeDisabledEntities]`
-   `[IncludePrefabs]`
-   `[IncludeSystemEntities]`
-   `[IncludeMetaEntities]`
-   `[UseWriteGroups]`

In addition, `AllowScheduleModes` can be used to restrict the types of modes
that the job supports scheduling for. Scheduling with an unsupported mode is
safety-checked at schedule time with a clear error message.

### Main Thread APIs

`IJobEach` has some unique capabilities compared to `IJobEntity`.

In addition to `Run()`, `Schedule()`, and `ScheduleParallel()` modes, it also
has a `RunImmediate()` mode which bypasses the job system completely. If you use
`RunImmediate()` outside of Burst, you can store managed fields inside your
`IJobEach`. Inside Burst, `RunImmediate()` often has less overhead than `Run()`.
`IJobEach` also supports all of the `*ByRef()` variants for scheduling.

There’s actually one method on `api` that wasn’t discussed.
`api.GetDefaultQuery<T>()` can be used to obtain the `EntityQuery` the job will
use if it is scheduled without one specified. You can use this to check if the
query is empty, or calculate the chunk or entity count in advance. The
`EntityQuery` will only be created and cached if you call this method, or
schedule the job without specifying an `EntityQuery`. `GetDefaultQuery<T>()` is
also how you add shared component and version filters to the `EntityQuery`.

If you’d like to provide your own `EntityQuery` instead, `FluentQuery` has the
`FromJob<T>()` method where you can pass in the `IJobEach`. It will populate the
`FluentQuery` with all requirements it would normally generate, including job
attributes.

### Injection

`IJobEach` supports the `[Inject]` attribute with the same rules of
`IInjectable`, except you don’t have to call `Inject()` on it. Injection happens
automatically at schedule time.

### Chunk Start and End

IJobEach has the following default interface methods:

```csharp
public bool OnChunkBegin(in JobContext context) => true;
public void OnChunkEnd(in JobContext context, bool chunkWasExecuted) {}
```

You can provide a custom implementation for one or both of these methods. The
protocol is the same as `IJobEntityChunkBeginEnd`, except most of the parameters
come from `IJobEach.JobContext` instead.

## Creating Your Own IParameter

`IParameter` is what completely unlocks `IJobEach` for extensibility. However,
writing your own is not trivial. Only do so when you feel the benefits outweigh
the costs.

The first thing you need to do is define the parameter type and its contents.
The type simply needs to implement `IParameter`, which has no methods to
implement.

```csharp
struct CircularBuffer : IJobEach.IParameter
{
    internal RefRW<CircularState> state;
    internal DynamicBuffer<CircularElement> buffer;
    // public Enqueue and Dequeue methods
    // …
}
```

### Parameter Handles

Next, you need to define a handle that produce these parameters. This type must
implement `IParameterHandle<T>` where `T` is your parameter type.
`IParameterHandle<T>` inherits `ILatiosApiGettable`. You must implement all
methods of `IParameterHandle<T>` including `ILatiosApiGettable` yourself.

A handle type can implement `IParameterHandle` for multiple different concrete
types at the same time. Additionally, both the `IParameter` and
`IParameterHandle` types can be generic.

The `IParameterHandle` interface provides start and end callbacks per chunk. It
also has a method for retrieving a parameter for a single entity index. This
means it is the responsibility of the handle to cache per-chunk data for the
subsequent calls. If the cached data involves a `NativeArray` or
`BufferAccessor`, it is strongly recommended to use `ThreadCache<T>` to store
the cached data. `ThreadCache<T>` avoids job safety issues.

```csharp
struct CircularBufferHandle : IJobEach.IParameterHandle<CircularBuffer>
{
    struct Cache
    {
        public NativeArray<CircularState>      states;
        public BufferAccessor<CircularElement> elements;
    }

    ComponentTypeHandle<CircularState> statesHandle;
    BufferTypeHandle<CircularElement>  elementsHandle;
    ThreadCache<Cache>                 threadCache;

    // This is called on a default instance whenever a query is being built
    public FluentQuery AppendToQuery(FluentQuery query)
    {
        return query.With<CircularState, CircularElement>(false);
    }

    public void CreateForApi(ref SystemState state)
    {
        statesHandle   = state.GetComponentTypeHandle<CircularState>(false);
        elementsHandle = state.GetBufferTypeHandle<CircularElement>(false);
    }

    public void UpdateForApi(ref SystemState state)
    {
        statesHandle.Update(ref state);
        elementsHandle.Update(ref state);
    }

    public bool OnChunkBegin(in IJobEach.JobContext context)
    {
        if (!threadCache.isCreated)
            threadCache = new ThreadCache<Cache>(new Cache());
        threadCache.cache.states   = context.chunk.GetNativeArray(ref statesHandle);
        threadCache.cache.elements = context.chunk.GetBufferAccessor(ref elementsHandle);
        return true;
    }

    public void OnChunkEnd(in IJobEach.JobContext context, bool chunkWasExecuted)
    {
    }

    public CircularBuffer GetParameter(in IJobEach.JobContext context)
    {
        return new CircularBuffer
        {
            state  = new RefRW<CircularState>(threadCache.cache.states, context.indexInChunk),
            buffer = threadCache.cache.elements[context.indexInChunk]
        };
    }
}
```

### Chunk Skipping Rules

Every `OnChunkBegin()` call allows a chunk to be discarded by returning `false`,
whether that be on the `IJobEach` directly or via an `IParameterHandle`. Once
one callback has returned `false`, no other callbacks are made. `IJobEach` has
`OnChunkBegin()` called first, which means it is always called. But an
`IParameterHandle` may never have an `OnChunkBegin()` called. Additionally, a
handle could have `OnChunkBegin()` called, but another parameter later down the
line discards the chunk.

`OnChunkEnd()` is only called for the handles that had their `OnChunkBegin()`
called. The callbacks occur in reverse order to the `OnChunkBegin()` callbacks.
This means that `IJobEach.OnChunkEnd()` is called last.

### Inserting Runtime Containers

It is **not safe** to access collection components from `LatiosWorldUnmanaged`
inside `ILatiosApiGettable.UpdateForApi()`. If a handle needs some temporary
container from the system’s `OnUpdate()`, the system should store the container
inside a normal `IComponentData` attached to the `SystemHandle`. Use
`EntityManager` or `BlackboardEntity` for this transaction. Do not use
`SystemAPI` or `ComponentLookup`.

### Registering IParameterHandle

The final step to make your `IParameter` usable is to register the
`IParameterHandle` for it. To do so, add the `[ParameterHandle]` attribute to
the `IParameter`. You must point the attribute to the `IParameterHandle` and
specify the scheduling modes the handle supports.

```csharp
[IJobEach.ParameterHandle(typeof(CircularBufferHandle), IJobEach.ScheduleModeMask.All)]
struct CircularBuffer : IJobEach.IParameter
{
    internal RefRW<CircularState> state;
    internal DynamicBuffer<CircularElement> buffer;
    // public Enqueue and Dequeue methods
    // …
}
```

An `IParameter` can have multiple `[ParameterHandle]` attributes. Additional
attributes must use the third constructor argument of `ParameterHandle`, which
is an array of `Execute()` method attribute types. A `ParameterHandle` is
considered a candidate if all of its `Execute()` method attribute types are
present on the `IParameter` argument in the job’s `Execute()` method. Among the
surviving candidates, the handle with the most matching `Execute()` method
attributes is selected.

For a generic handle type, specify an open generic for the type:
`typeof(CustomGenericHandle<>)`

## Grouping Chunks

In addition to `IParameterHandle<T>`, there is also an
`IGroupedParameterHandle<T>`. This handle type is used to force multiple chunks
to be processed on the same thread. Some data structures may require chunks be
grouped by shared component, spatial hash, or hierarchy.

`IGroupedParameterHandle` requires 3 additional methods be implemented in
addition to all the `IParameterHandle` methods.

### Unioning

The first is `ScheduleGroupedUnions()`. This method is **only called** in
`ScheduleParallel()` mode. The method is given a `ChunkGroupBuilder` and a
`JobHandle`, and is expected to schedule a single-threaded `IJob` to group
chunks together.

Inside this `IJob`, `ChunkGroupBuilder` provides `NativeList<CapturedChunk>
chunks`, and `NativeList<ChunkGroupRange> groupRanges`. A `CapturedChunk`
contains the chunk as well as other job metadata such as `IEnableableComponent`
filtering and the `entityIndexInQuery` prefix sum. A `ChunkGroupRange` specifies
ranges in `chunks` that have already been grouped together. For explanatory
purposes, the index of each `ChunkGroupRange` within `groupRanges` will be
referred to as the `GroupID`.

The `IJob` should iterate the chunks, and identify pairs of chunks need to
belong to the same group. For each chunk in the pair, it must identify the
chunk’s `GroupID`. Then the job must call `Union()` on the `ChunkGroupBuilder`
passing in the two `GroupID`s. `Union()` does not modify the `NativeList`s in
any way. Instead, `ChunkGroupBuilder` maintains an internal record of edits.
Once all analysis is complete, call `ApplyChanges()` on the `ChunkGroupBuilder`
to update the two lists. Here’s an example for grouping by shared component:

```csharp
struct GroupTeamsJob : IJob
{
    public IJobEach.ChunkGroupBuilder builder;
    public SharedComponentTypeHandle<Team> teamHandle;

    public void Execute()
    {
        var teamToFirstGroupMap = new UnsafeHashMap<Team, int>(builder.groupRanges.Length, Allocator.Temp);

        for (int groupId = 0; groupId < builder.groupRanges.Length; groupId++)
        {
            var group = builder.groupRanges[groupId];
            for (int i = 0; i < group.count; i++)
            {
                var chunk = builder.chunks[group.start + i];
                var team = chunk.chunk.GetSharedComponent(teamHandle);
                if (teamToFirstGroupMap.TryGetValue(team, out int firstGroupID))
                {
                    builder.Union(firstGroupID, groupId);
                }
                else
                {
                    teamToFirstGroupMap.Add(team, groupId);
                }
            }
        }
        builder.ApplyChanges();
    }
}
```

### Processing Groups

The other two methods of `IGroupedParameterHandle` are `OnGroupBegin()` and
`OnGroupEnd()`. These methods are called for every `IGroupedParameterHandle` for
every group. The methods pass in a `GroupContext`, which provides a limited set
of metadata about the job and group. For all schedule modes except
`ScheduleParallel()`, all chunks are forced into a single group.

### An Example Use Case

Using the `Team` grouping example above, imagine if every team had a captain
entity, and many member entities. All the member entities need to report to
their captain. Normally, a `ComponentLookup` is not safe to write to in
parallel. Member entities could span across different chunks and try to report
to their captain from multiple threads at the same time. With grouping, all
member entities will be guaranteed to be processed on the same thread. And
therefore, the `ComponentLookup` is safe. The `IGroupedParameterHandle` can
query all the unique teams and their captains during `UpdateForApi()`. Then it
can use `[NativeDisableParallelForRestriction]` on the `ComponentLookup`s. In
`OnChunkBegin()`, it looks up the captain of the team, and saves `RefRW`s and
`DynamicBuffer`s inside a `ThreadCache`. It passes these into the `IParameter`
it provides.

With this, a user can simply specify they want the `CaptainData` `IParameter` in
the `Execute()` method, and not worry about anything else.
