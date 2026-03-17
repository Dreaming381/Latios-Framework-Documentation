# QVVS V2 Upgrade Guide

QVVS Transforms V2 marks a big transition to how transforms operate within an
ECS project. Upgrading an existing project may be challenging. Hopefully, this
guide will help ease that burden.

## Paradigm Shift

QVVS V2 is an always-up-to-date transform system. That means that the transform
hierarchy is in-sync at all times. If you modify a parent’s position, the
child’s world-space position will be updated immediately. This works on the main
thread, and in jobs. There’s no more `TransformSuperSystem` responsible for
synchronizing everything. There’s no deferring of updates. Everything happens
on-the-spot, and if your game logic expected otherwise, then it needs to be
updated.

This also changes the performance characteristics and optimization techniques.
Writing to a `TransformAspect` can be a lot more expensive. However, choosing to
only update root transforms of some entities at a lower frequency is a much more
effective optimization than it was in V1.

Lastly, this means that the `Disabled` component has no effect on transform
propagation. If an entity’s transform is modified, its disabled child will still
be updated accordingly.

## Authoring

**Live Baking in Play Mode and modifying transforms from an EditorWorld system
are NOT supported.** This means you should go into play mode, and then move
transforms around in the scene view or inspector. You also shouldn’t use systems
in the editor world which modify transforms in any way (an animation previewer
would be an example of such a system).

Sometimes, things will work anyways, and sometimes things will break, because
Unity’s live patcher has no clue what is going on. Perhaps someday I will make
live baking play nice, but *a lot* of code needs to be written to handle that.

As for writing custom bakers, avoid `TransformUsageFlags.ManualOverride` if at
all possible, as it is extremely bug-prone. You can make use of the new
`BakedLocalTransformOverride` and `BakedParentOverride` components for most use
cases.

Also, `IRequestCopyParentTransform` is no longer present, as that is now an
inheritance flag.

Unlike in V1, the sibling order of Game Objects are preserved in the runtime
hierarchy in V2.

## Writing to Components

Some of the runtime components remain the same, such as `WorldTransform`,
`PreviousTransform`, and `TwoAgoTransform`. The rest are different. In V1, you
were generally discouraged from writing to `WorldTransform` and `LocalTransform`
unless you were instantiating entities. **In V2, you should never set or write
to any runtime transform component directly, even when instantiating entities.**

You can use `TransformAspect` for all writing needs. And when instantiating
entities in a job, you can make use of `WorldTransformCommand`,
`ParentTransformCommand`, and `ParentWithLocalTransformCommand` to initialize
transforms of entities.

Also, be very wary about removing entities from `LinkedEntityGroup`. Doing so
risks safety-check exceptions or crashes in builds.

If for some reason, you want to write transforms while avoiding
`TransformAspect`, you can use the lower-level `TransformTools` APIs.

## Instantiation and Lifecycle Rules

In general, if you instantiate prefab entities, you shouldn’t have any issues.
Also, if you instantiate a root or solo entity, things should go smoothly. But
if you try to instantiate a live child entity, only in a limited number of
circumstances will you not receive an error message. If you find your use case
isn’t supported, please reach out on discord!

Instantiating via `EntityCommandBuffer` is generally not recommended, as you
can’t initialize transform values with it anymore. Instead, use
`InstantiateCommandBuffer` and the appropriate `IInstantiateCommands`.

Unlike in V1 where the transform system was independent of `LinkedEntityGroup`,
V2 tries to keep `LinkedEntityGroup` in-sync with the root hierarchy. There’s a
big performance speed-up when a root can assume that if it is destroyed, all
descendants will be destroyed as well.

## Parenting Rules

In V1, if a parent entity is destroyed, the children become root entities. In
V2, ancestors inherit the children of dead parents, unless the dead parent is a
root. If the dead parent is a root, usually all the children will be destroyed
as well due to belonging to that dead root’s `LinkedEntityGroup`. But if the
children were added to the hierarchy at runtime without `LinkedEntityGroup`
attachment, then the root entity will use a cleanup component to stay alive and
preserve the hierarchy until there are no more alive descendants.

You can change parents at runtime using the `EntityManager` extension methods
`SetParent()` and `ClearParent()`. Altering parents in a job is not supported.

## InheritanceFlags

`HierarchyUpdateMode.Flags` in V1 has been renamed to `InheritanceFlags` in V2.
There is no longer a component you can set on the fly. Instead, the flags live
inside the hierarchy buffers. `CopyParentWorldTransformTag` no longer exists,
and is instead replaced by `InheritanceFlags.CopyParent`.

## Reading and Navigating Hierarchies

When a root entity is instantiated, its full hierarchy info is already set up
and ready to navigate. Navigating hierarchies is much faster than in V1.

Every entity with a transform will have the `WorldTransform` component, which
you can read directly whenever. If the entity is an alive root entity with
children, then it will have the dynamic buffer `EntityInHierarchy`. If the
entity is not a root entity, but has a parent, then it will have the
`RootReference` component.

From a `RootReference` instance, you can call `ToHandle()` to get an
`EntityInHierarchyHandle`. `EntityInHierarchyHandle` allows you to discover
other entities in the hierarchy via their relationships. Note that some entities
in the hierarchy may be destroyed. You must call one of `SetParent()`,
`ClearParent()`, or `CleanHierarchy()` methods on the `EntityManager` to remove
destroyed entities from the hierarchy.

There is no `LocalTransform` component. Instead, use
`TransformTools.LocalTransformFrom()` if you need to read the local transform in
a read-only context.

## TransformAspect

`TransformAspect` is no longer an `IAspect`. How you acquire it has changed. But
once you successfully acquire it, it is a lot more powerful than before.

QVVS V2 is designed such that obtaining a `TransformAspect` gives you write
access to all the transforms in the hierarchy. You can use the
`entityInHierarchyHandle` property to discover other entities in the hierarchy.
And then you can use the `[]` indexer with any other `entityInHierarchyHandle`
from the same hierarchy to get that other entity’s `TransformAspect`.

Keep in mind that setting `TransformAspect` properties can be much more
expensive, as each assignment might trigger an update of all descendants of that
entity.

## Obtaining TransformAspects

From the main thread, you can obtain a `TransformAspect` using the
`EntityManager` extension method `GetTransformAspect()`. `ComponentBroker` has a
similar API.

In a job, there are multiple special handle types you can use.
`TransformAspectLookup` is the most straightforward. **Never use**
`[NativeDisableParallelForRestriction]` **with** `TransformAspectLookup`**!**

Use the following snippet to construct it inside a system’s `OnUpdate()` method:

```csharp
new TransformAspectLookup(SystemAPI.GetComponentLookup<WorldTransform>(false),
                       SystemAPI.GetComponentLookup<RootReference>(true),
                       SystemAPI.GetBufferLookup<EntityInHierarchy>(true),
                       SystemAPI.GetBufferLookup<EntityInHierarchyCleanup>(true),
                       SystemAPI.GetEntityStorageInfoLookup())
```

You can always find snippets like this by following the type’s declaration in
your IDE.

## Obtaining Root TransformAspects in Parallel

If you know you are going to be iterating roots and want write access, you can
use `TransformAspectRootHandle`, constructed as follows:

```csharp
new TransformAspectRootHandle(SystemAPI.GetComponentLookup<WorldTransform>(false),
                           SystemAPI.GetBufferTypeHandle<EntityInHierarchy>(true),
                           SystemAPI.GetBufferTypeHandle<EntityInHierarchyCleanup>(true),
                           SystemAPI.GetEntityStorageInfoLookup())
```

For each chunk, you must call `SetupChunk()` before using its `[]` accessor.
Make sure to include `WorldTransform` in your query. Here’s an example of how to
use this API in an `IJobEntity`:

```csharp
[BurstCompile]
[WithAll(typeof(BulletTag), typeof(WorldTransform))]
partial struct Job : IJobEntity, IJobEntityChunkBeginEnd
{
    public TransformAspectRootHandle transformHandle;
    public float                     dt;

    public void Execute([EntityIndexInChunk] int indexInChunk, in Speed speed)
    {
        var transform            = transformHandle[indexInChunk];
        transform.worldPosition += dt * speed.speed * transform.forwardDirection;
    }

    public bool OnChunkBegin(in ArchetypeChunk chunk, int unfilteredChunkIndex, bool useEnabledMask, in v128 chunkEnabledMask)
    {
        transformHandle.SetupChunk(in chunk);
        return true;
    }

    public void OnChunkEnd(in ArchetypeChunk chunk, int unfilteredChunkIndex, bool useEnabledMask, in v128 chunkEnabledMask, bool chunkWasExecuted)
    {
    }
}
```

## Obtaining Arbitrary TransformAspects in Parallel

While not as fast as `TransformAspectRootHandle`, you can use
`TransformAspectParallelChunkHandle` if you need to write arbitrary transforms
in parallel. This type requires a little more complex setup compared to the
others.

*Warning: While this API updates entities within the same hierarchy in a
simulation-wide deterministic order, that order is based on the order of
entities in the query, which is usually hard to predict.*

Using the API requires scheduling 3 separate jobs. The first job collects the
entities and chunks from the `EntityQuery`. The second job groups chunks that
need to update on the same thread for safety. And the final job actually
processes the entities with your job code.

### IJobEntity Example:

For `IJobEntity`, the `IJobEntity` job itself is used for the collection phase.
This is done by returning the value of
`TransformAspectParallelChunkHandle.OnChunkBegin()` inside the
`IJobEntityChunkBeginEnd`’s interface. This call will return `false` during the
collection phase, and `true` during the final processing phase.

```csharp
[BurstCompile]
public void OnUpdate(ref SystemState state)
{
    var job = new Job
    {
        transformHandle = new TransformAspectParallelChunkHandle(SystemAPI.GetComponentLookup<WorldTransform>(false),
                                                                 SystemAPI.GetComponentTypeHandle<RootReference>(true),
                                                                 SystemAPI.GetBufferLookup<EntityInHierarchy>(true),
                                                                 SystemAPI.GetBufferLookup<EntityInHierarchyCleanup>(true),
                                                                 SystemAPI.GetEntityStorageInfoLookup(),
                                                                 ref state)
    };
    job.ScheduleByRef(); // You MUST use ScheduleByRef() specifically here. IJobEntity.Execute() does NOT run here.
    state.Dependency = job.transformHandle.ScheduleChunkGrouping(state.Dependency);
    state.Dependency = job.GetTransformsScheduler().ScheduleParallel(state.Dependency); // IJobEntity.Execute() actually runs here.
}

[WithAll(typeof(WorldTransform))]
[BurstCompile]
partial struct Job : IJobEntity, IJobEntityChunkBeginEnd, IJobChunkParallelTransform
{
    public TransformAspectParallelChunkHandle transformHandle;

    public ref TransformAspectParallelChunkHandle transformAspectHandleAccess => ref transformHandle.RefAccess();

    public void Execute([EntityIndexInChunk] int indexInChunk, in TimeToLive timeToLive, in SpawnPointAnimationData data)
    {
        float growFactor   = math.unlerp(data.growStartTime, data.growEndTime, timeToLive.timeToLive);
        growFactor         = math.select(growFactor, 1f, data.growStartTime == data.growEndTime);
        float shrinkFactor = math.unlerp(0f, data.shrinkStartTime, timeToLive.timeToLive);
        float factor       = math.saturate(math.min(growFactor, shrinkFactor));
        bool  isGrowing    = growFactor < shrinkFactor;

        float growRadians   = math.lerp(-data.growSpins, 0f, factor);
        float shrinkRadians = math.lerp(data.shrinkSpins, 0f, factor);
        float rads          = math.select(shrinkRadians, growRadians, isGrowing);

        var transform           = transformHandle[indexInChunk];
        transform.localRotation = quaternion.Euler(0f, 0f, rads);
        transform.localScale    = math.max(factor, 0.001f);
    }

    public bool OnChunkBegin(in ArchetypeChunk chunk, int unfilteredChunkIndex, bool useEnabledMask, in v128 chunkEnabledMask)
    {
        return transformHandle.OnChunkBegin(in chunk, unfilteredChunkIndex, useEnabledMask, chunkEnabledMask);
    }

    public void OnChunkEnd(in ArchetypeChunk chunk, int unfilteredChunkIndex, bool useEnabledMask, in v128 chunkEnabledMask, bool chunkWasExecuted)
    {
    }
}
```

### IJobChunk

The only difference between `IJobEntity` and `IJobChunk` is that you need to
call `OnChunkBegin()` at the beginning of `Execute()`, and manually check the
return value like so:

```csharp
// If using ScheduleByRef()
if (!transformHandle.OnChunkBegin(in chunk, unfilteredChunkIndex, useEnabledMask, chunkEnabledMask))
    return;

// If using ScheduleChunkCaptureForQuery() - though the above will work too
transformHandle.OnChunkBegin(in chunk, unfilteredChunkIndex, useEnabledMask, chunkEnabledMask);
```

Optionally, you can make the following substitution:

```csharp
// Instead of this:
job.ScheduleByRef(m_query);
// You can do this:
state.Dependency = job.transformHandle.ScheduleChunkCaptureForQuery(m_query, state.Dependency);
```

## Other Noteworthy APIs

QVVS V2 has a few more APIs you might want to try and take advantage of.

### TransformsComponentLookup and TransformsBufferLookup

Since `TransformAspect` gives you exclusive access to all transforms in the
hierarchy, why can’t you also get exclusive access to other components in the
hierarchy as well?

If you have used Psyshock, you may already be familiar with the
`PhysicsComponentLookup` and `PhysicsBufferLookup` types. These work the same,
except instead of a `SafeEntity`, you need a `TransformsKey` to access them. You
can get this key via the `transformsKey` property of
`TransformAspectRootHandle`, and by
`TransformAspectParallelChunkHandle.GetTransformsKey()`. If you have exclusive
access to a solo or root entity, you can also construct a `TransformsKey`. It is
safe to do this for a solo or root `SafeEntity` from Psyshock.

### IJobParallelForDefer Mode of TransformAspectParallelChunkHandle

`TransformAspectParallelChunkHandle` has an extra mode of operation, which
allows you to schedule an `IJobParallelForDefer` job. This allows a single
`Execute()` method to process all the chunks that need to be batched together at
once, so that you can reason about all the entities in a hierarchy that match an
`EntityQuery`. Instead of `OnChunkBegin()`, you call
`GetChunkCountForIJobParallelForDeferIndex()`,
`GetChunkInGroupForIJobParallelForDefer()`, and
`SetActiveChunkForIJobParallelForDefer()` to iterate and access the chunks in
the batch.

### TransformBatchWriteCommand

`TransformBatchWriteCommand` is an API that allows you to write to multiple
`TransformAspect`s of the same hierarchy in one operation. This can be
significantly more efficient than writing to each `TransformAspect` one-by-one,
as hierarchy propagation is only performed once.

## Missing Something?

QVVS V2 is a new solution, and has room to grow. If you find you are missing the
APIs you need for your use case, or you having trouble meeting performance
targets with the new solution, please report these challenges! That way,
resolutions can be prioritized.
