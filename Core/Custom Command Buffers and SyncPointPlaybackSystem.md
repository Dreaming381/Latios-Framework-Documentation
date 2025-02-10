# Custom Command Buffers and SyncPointPlaybackSystem

Sync points due to structural changes in DOTS are very expensive. During this
period, the world stops and the worker threads go idle wasting performance while
the main thread computes the structural changes. Latios Framework aims to
minimize the performance loss of these sync points by providing custom command
buffers which expand the available range of performance and features, and by
providing a dedicated sync point which permits some types of jobs to run on the
worker threads instead of sitting idle.

## Command Buffers

Latios Framework provides five custom command buffers. These command buffers are
significantly more limited than `EntityCommandBuffer` in that they each service
only one type of command. All command buffers provide a deterministic
`ParallelWriter` instance in which commands require a `sortKey` argument. The
command buffers provide proper safety checks to prevent misuse. `Add()` is
always the method for writing a new command. They can be used with custom
allocators and utilized fully in Burst-compiled unmanaged systems.

### EnableCommandBuffer

`EnableCommandBuffer` allows enabling an entity as if it had
`EntityManager.SetEnabled(entity, true)` invoked on it. This means that entities
in the hierarchy (`LinkedEntityGroup`) are also enabled. The hierarchy is
captured during playback rather than when the commands were recorded.

To manually play back an `EnableCommandBuffer`, you must provide both an
`EntityManager` and a read-only `BufferFromEntity<LinkedEntityGroup>`. You must
dispose the command buffer after manual playback.

Example:

```csharp
[BurstCompile]
public void OnUpdate(ref SystemState state)
{
    var ecb = new EnableCommandBuffer(Allocator.TempJob);

    foreach ((var payload, var times, var entity) in Query<RefRW<SpawnPayload>, RefRO<SpawnTimes> >().WithEntityAccess())
    {
        if (times.ValueRO.enableTime <= 0f && payload.ValueRO.disabledShip != Entity.Null)
        {
            var ship = payload.ValueRO.disabledShip;
            ecb.Add(ship);

            var entityTransform          = GetComponent<WorldTransform>(entity);
            var shipTransform            = GetAspect<TransformAspect>(ship);
            shipTransform.worldRotation  = entityTransform.rotation;
            shipTransform.worldPosition  = entityTransform.position;
            payload.ValueRW.disabledShip = Entity.Null;
        }
    }

    ecb.Playback(state.EntityManager, GetBufferLookup<LinkedEntityGroup>(true));
    ecb.Dispose();
}
```

### DisableCommandBuffer

`DisableCommandBuffer` is identical to `EnableCommandBuffer` except it
represents invoking `EntityManager.SetEnabled(entity, false)`.

### DestroyCommandBuffer

`DestroyCommandBuffer` is capable of destroying entities and may provide a
performance improvement over `EntityCommandBuffer` in some circumstances as
entities are destroyed using batch processing.

To manually play back a `DestroyCommandBuffer`, you must provide an
`EntityManager`. After manual playback, you must dispose the command buffer.

### InstantiateCommandBuffer

`InstantiateCommandBuffer` can instantiate an entity, add or set up to five
components on it with values, and add an additional fifteen component types. The
types of components added or set is identical for all commands in the command
buffer.

The generic arguments represent the components you wish to initialize with
values. If a component does not exist on the entity, it will be added.
Otherwise, the prefab’s default value for that component will be overwritten. A
non-generic variant also exists when no initialization is required.

You can add additional components using the `AddComponentTag()` method. You may
also use `SetComponentTags()` to add multiple tags at once. If the prefab
already has one of these components, the already existing component will be left
untouched.

*Important: The tag components are container-wide and are applied to all root
entities instantiated.*

The original prefabs passed into the commands will be left untouched.

To manually play back an `InstantiateCommandBuffer`, you must provide an
`EntityManager`. After manual playback, you must dispose the command buffer.

`InstantiateCommandBuffer`’s purpose is primarily performance. It uses batching
both in instantiation and value initialization, frequently outperforming
`EntityCommandBuffer` for such usages. It also generates far fewer archetypes to
reach the end result.

Example:

```csharp
[BurstCompile]
public void OnUpdate(ref SystemState state)
{
    var icb = latiosWorld.syncPoint.CreateInstantiateCommandBuffer<WorldTransform>().AsParallelWriter();
    var dcb = latiosWorld.syncPoint.CreateDestroyCommandBuffer().AsParallelWriter();

    new Job { dcb = dcb, icb = icb }.ScheduleParallel();
}

[BurstCompile]
[WithChangeFilter(typeof(ShipHealth))]
partial struct Job : IJobEntity
{
    public InstantiateCommandBuffer<WorldTransform>.ParallelWriter icb;
    public DestroyCommandBuffer.ParallelWriter                     dcb;

    public void Execute(Entity entity,
                        [ChunkIndexInQuery] int chunkIndexInQuery,
                        in ShipHealth health,
                        in ShipExplosionPrefab explosionPrefab,
                        in WorldTransform worldTransform)
    {
        if (health.health <= 0f)
        {
            dcb.Add(entity, chunkIndexInQuery);
            if (explosionPrefab.explosionPrefab != Entity.Null)
                icb.Add(explosionPrefab.explosionPrefab, worldTransform, chunkIndexInQuery);
        }
    }
}
```

### AddComponentsCommandBuffer

`AddComponentsCommandBuffer` is similar to `InstantiateCommandBuffer`, except
without the instantiation part. It can initialize up to 5 components, and add up
to 15 more.

Additionally, `AddComponentsCommandBuffer` has an
`AddComponentsDestroyedEntityResolution` argument in its constructor, which
allows you to specify what to do when playback determines the target entity has
been destroyed. You can choose to `DropData` and skip the command, you can
choose `ThrowException` to alert the presence of a bug, or you can choose
`AddToNewEntityAndDestroy`, in which a new entity will be created, have the
components added to, and then promptly destroyed. The lattermost option is
useful for ensuring cleanup component values are preserved to avoid potential
memory leaks or other cleanup-related problems.

### EntityOperationCommandBuffer

In the case where you need to store only the entity in some command buffer
operation, whether it be for a structural change operation or some other
purpose, you may wish to use `EntityOperationCommandBuffer`. This is a
safety-checks compliant `[NativeContainer] `which powers
`Enable/Disable/DestroyCommandBuffer` as well as the non-generic
`InstantiateCommandBuffer`.

You can add entities to it in a single or parallel job, but in a single-threaded
job, you can also have those entities read back to you in sorted order. You can
sort exclusively by `sortKey` or have identical entities grouped together. There
is also API for fetching `LinkedEntityGroup` entities.

### For Those with Concerns

It may be alarming that these custom command buffers exist which promise either
new features or better performance when performing structural changes. One might
suspect that these command buffers are taking shortcuts which could potentially
corrupt Unity’s internal data structures and easily break between DOTS releases.

Rest assured; these command buffers rely entirely on public API. The new
features and performance improvements come from clever operations and a powerful
backing data structure. You can read more about the design in Optimization
Adventures 4.

## SyncPointPlaybackSystem

The `SyncPointPlaybackSystem` is one of the systems automatically created by the
[LatiosWorld](LatiosWorld%20in%20Detail.md). By default, it executes before
`BeginInitializationEntityCommandBufferSystem`. It is capable of playing back
the following command buffers:

-   EntityCommandBuffer
-   EnableCommandBuffer
-   DisableCommandBuffer
-   DestroyCommandBuffer
-   InstantiateCommandBuffer
-   AddComponentsCommandBuffer

It will play back these buffers in the order they are requested from systems.
The system is an unmanaged system invoked from a managed
`SyncPointPlaybackSystemDispatch` which is responsible for catching exceptions
and in such situations reissuing updates until all command buffers are
processed.

### Creation and Usage

To request a command buffer, call the appropriate `Create*CommandBuffer()`
method.

If you request a command buffer from somewhere other than a Latios Framework
dispatched system, you must also call `AddJobHandleForProducer()` or
`AddMainThreadCompletionForProducer()`.

Like `EntityCommandBufferSystem`, `SyncPointPlaybackSystem` will capture the
requesting system for each command buffer and add this to the profiling metadata
during playback.

### Simulation Sync Points

The early frame sync point location was chosen to avoid unnecessary overhead for
some game loops while also providing an opportunity to run special jobs on the
worker threads. However, for situations where a sync point is required inside
`LatiosSimulationSystemGroup`, rather than try to create a second instance of
`SyncPointPlaybackSystem`, it is recommended to explicitly add the system to a
[SuperSystem](Super%20Systems.md) or [RootSuperSystem](Super%20Systems.md). You
may need to set `EnableSystemSorting` to false.

This does have the caveat that command buffers will play back at the next
instance the `SyncPointPlaybackSystem` executes, so make sure you are fully
aware of your system order if there is concern for a command buffer to play back
too early. Note that `SyncPointPlaybackSystem` is a Burst-compiled `ISystem`
that is dispatched from a managed `SyncPointPlaybackSystemDispatch`. The latter
is what you want to add to other `ComponentSystemGroups`.

## PreSyncPointGroup

During a sync point, the full ECS world goes into lockdown on the main thread.
No job is allowed to touch any components. Component data arrays from entity
queries fall out of sync. And with the possibility of entities being created and
destroyed, it is difficult to identify ECS simulation work that can adapt to
such structural changes and still be worth the additional random-access copy
overhead before and after the sync point.

However, that isn’t to say that the worker threads must sit idle during this
period. There are certainly a few tasks which are not affected by lockdown.
Typically, these are operations that have IO-like behavior. Here are some
examples which may or may not be valid or applicable depending on your project:

-   Audio
-   Game stats crunching
-   Procedural geometry – especially useful for n-1 rendering
-   Procedural CPU textures – especially useful for n-1 rendering
-   File compression/decompression
-   Network message packing/unpacking

For systems which are capable of scheduling such jobs, it is recommended to
update them in `PreSyncPointGroup`. As its name implies, this group runs at the
very beginning of the frame right before
`BeginInitializationEntityCommandBufferSystem` and `SyncPointPlaybackSystem`
execute.

### Preventing Jobs from Completing Early

A challenge when scheduling these jobs is preventing the sync point from trying
to complete them. For that, these jobs cannot be assigned to
`SystemState.Dependency`.

1.  Allocate NativeContainers to store captured ECS state
2.  Schedule ECS jobs which capture the state
3.  Ensure `Dependency` represents all ECS jobs
4.  Copy `Dependency` into a local `JobHandle` variable
5.  Schedule non-ECS jobs that should run during the sync point using the new
    local `JobHandle` variable
6.  Store the local `JobHandle` variable somewhere safe

Regarding the final step, where to store the `JobHandle` depends on where the
data is consumed.

For audio, file writing, and network transmission, typically only the producing
system cares that the jobs are completed. Consequently, the system can store the
`JobHandle` in the field and complete it at the beginning of its next
`OnUpdate()` invocation, as well as in `OnDestroy()`.

For other use cases, the `JobHandle` needs to be exported from the system so
that other systems can read the results. A good candidate for this is a
collection component on the `worldBlackboardEntity`. The `worldBlackboardEntity`
should only be destroyed when the ECS World is also being destroyed, in which
case the framework will ensure the job is completed before teardown. And it is
also easy enough to avoid removing the collection component from the
`worldBlackboardEntity` during sync by establishing conventions (but if you do,
the job will still be completed properly rather than throw an exception).

There are other methods for handling this `JobHandle`, but these two usually
account for all use cases and leverage the strengths of the framework.
