# Writing QVVS Transforms in Parallel

QVVS Transforms is always up-to-date, and that means that writing to the
transform of one entity potentially requires the transforms of other entities to
also update. And if those entities are being processed by different threads at
the same time, race conditions occur.

This is actually quite a big challenge, and most engines and ECS solutions opt
to avoid it altogether. QVVS Transforms aims to solve it head on with clever API
design that prevents race conditions from happening.

## Iterating Roots

In order to write transforms in parallel, we have to guarantee that two threads
don’t have access to two entities from the same hierarchy at the same time.

One easy way to guarantee that is to simply not iterate entities that have
parents. This is what the `TransformAspectRootHandle` does. To use it, you will
need an `EntityQuery` that includes `WorldTransform` and excludes the
`RootReference` component.

You can construct a handle inside the `OnUpdate()` of a system like this:

```csharp
new TransformAspectRootHandle(SystemAPI.GetComponentLookup<WorldTransform>(false),
                           SystemAPI.GetBufferTypeHandle<EntityInHierarchy>(true),
                           SystemAPI.GetBufferTypeHandle<EntityInHierarchyCleanup>(true),
                           SystemAPI.GetEntityStorageInfoLookup())
```

You can then pass that into an `IJobChunk` or `IJobEntity`. Inside the job, you
need to initialize the handle with the chunk you want to process. Then you can
index the handle using the entity index in the chunk.

Here’s what a simple `IJobChunk` job looks like:

```csharp
[BurstCompile]
struct ChunkRootTransformsJob : IJobChunk
{
    public TransformAspectRootHandle transformHandle;

    public void Execute(in ArchetypeChunk chunk, int unfilteredChunkIndex, bool useEnabledMask, in v128 chunkEnabledMask)
    {
        transformHandle.SetupChunk(in chunk);

        var enumerator = new ChunkEntityEnumerator(useEnabledMask, chunkEnabledMask, chunk.Count);
        while (enumerator.NextEntityIndex(out int entityIndexInChunk))
        {
            TransformAspect transform = transformHandle[entityIndexInChunk];
            transform.TranslateWorld(math.forward() * 0.01f);
        }
    }
}
```

And here is the same job as an `IJobEntity`:

```csharp
[BurstCompile]
[WithAll(typeof(WorldTransform))]
[WithNone(typeof(RootReference))]
partial struct EntityRootTransformsJob : IJobEntity, IJobEntityChunkBeginEnd
{
    public TransformAspectRootHandle transformHandle;

    public bool OnChunkBegin(in ArchetypeChunk chunk, int unfilteredChunkIndex, bool useEnabledMask, in v128 chunkEnabledMask)
    {
        transformHandle.SetupChunk(in chunk);
        return true;
    }

    public void OnChunkEnd(in ArchetypeChunk chunk, int unfilteredChunkIndex, bool useEnabledMask, in v128 chunkEnabledMask, bool chunkWasExecuted)
    {
    }

    public void Execute([EntityIndexInChunk] int indexInChunk)
    {
        var transform = transformHandle[indexInChunk];
        transform.TranslateWorld(transform.forwardDirection * 0.01f);
    }
}
```

This API is optimized for speed, and has nearly no overhead compared to directly
iterating `WorldTransform` on solo entities.

## Iterating Arbitrary Entities

A second way to guarantee that two threads don’t access two entities from the
same hierarchy is to group chunks together which share entities from the same
hierarchy. Then each thread can process a full group of chunks.

The whole process works in three steps:

1.  Collect the chunks, including filtering info
2.  Group the chunks
3.  Process the groups of chunks

This is what `TransformAspectParallelChunkHandle` offers. Steps 1 & 2 are
single-threaded, but are usually quite fast compared to step 3. Step 3 is the
parallel job that is actually writing transforms.

For this to work, you will need an `EntityQuery` that includes `WorldTransform`.
Then in `OnUpdate()` you will need to create the
`TransformAspectParallelChunkHandle` and schedule the three jobs in order. For
the third job, you actually schedule a wrapper job around either an `IJobEntity`
or `IJobChunk`. This way, you are working with a familiar job interface.
However, that wrapper needs access to the `TransformAspectParallelChunkHandle`,
so your job also needs to implement the `IJobChunkParallelTransform` interface.

This is what a system looks like using `IJobChunk`:

```csharp
partial struct ChunkTransformsSystem : ISystem
{
    EntityQuery m_query;

    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        m_query = state.Fluent().With<WorldTransform>().Build();
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var transformHandle = new TransformAspectParallelChunkHandle(SystemAPI.GetComponentLookup<WorldTransform>(false),
                                                                     SystemAPI.GetComponentTypeHandle<RootReference>(true),
                                                                     SystemAPI.GetBufferLookup<EntityInHierarchy>(true),
                                                                     SystemAPI.GetBufferLookup<EntityInHierarchyCleanup>(true),
                                                                     SystemAPI.GetEntityStorageInfoLookup(),
                                                                     ref state);

        state.Dependency = transformHandle.ScheduleChunkCaptureForQuery(m_query, state.Dependency);
        state.Dependency = transformHandle.ScheduleChunkGrouping(state.Dependency);
        var job          = new ChunkJob { transformHandle = transformHandle };
        state.Dependency = job.GetTransformsScheduler().ScheduleParallel(state.Dependency);
    }

    struct ChunkJob : IJobChunk, IJobChunkParallelTransform
    {
        public TransformAspectParallelChunkHandle transformHandle;

        ref TransformAspectParallelChunkHandle IJobChunkParallelTransform.transformAspectHandleAccess => ref transformHandle.RefAccess();

        public void Execute(in ArchetypeChunk chunk, int unfilteredChunkIndex, bool useEnabledMask, in v128 chunkEnabledMask)
        {
            transformHandle.OnChunkBegin(in chunk, unfilteredChunkIndex, useEnabledMask, in chunkEnabledMask);

            var enumerator = new ChunkEntityEnumerator(useEnabledMask, chunkEnabledMask, chunk.Count);
            while (enumerator.NextEntityIndex(out int entityIndexInChunk))
            {
                TransformAspect transform = transformHandle[entityIndexInChunk];
                transform.TranslateWorld(transform.forwardDirection * 0.01f);
            }
        }
    }
}
```

Using `IJobEntity` is a little trickier, due to the way Unity’s source
generators work. We need to trick Unity into performing dependency injection on
a struct that we can then wrap. The way we do this is by using the `IJobEntity`
for Step 1, and explicitly using `ScheduleByRef()`. This is what the same system
as above looks like, but using `IJobEntity`. Note the `ScheduleByRef()`, and the
usage of the return value of `OnChunkBegin()`.

```csharp
partial struct EntityTransformSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var job = new EntityJob
        {
            transformHandle = new TransformAspectParallelChunkHandle(SystemAPI.GetComponentLookup<WorldTransform>(false),
                                                                     SystemAPI.GetComponentTypeHandle<RootReference>(true),
                                                                     SystemAPI.GetBufferLookup<EntityInHierarchy>(true),
                                                                     SystemAPI.GetBufferLookup<EntityInHierarchyCleanup>(true),
                                                                     SystemAPI.GetEntityStorageInfoLookup(),
                                                                     ref state)
        };
        job.ScheduleByRef();
        state.Dependency = job.transformHandle.ScheduleChunkGrouping(state.Dependency);
        state.Dependency = job.GetTransformsScheduler().ScheduleParallel(state.Dependency);
    }

    [WithAll(typeof(WorldTransform))]
    [BurstCompile]
    partial struct EntityJob : IJobEntity, IJobEntityChunkBeginEnd, IJobChunkParallelTransform
    {
        public TransformAspectParallelChunkHandle transformHandle;

        ref TransformAspectParallelChunkHandle IJobChunkParallelTransform.transformAspectHandleAccess => ref transformHandle.RefAccess();

        public bool OnChunkBegin(in ArchetypeChunk chunk, int unfilteredChunkIndex, bool useEnabledMask, in v128 chunkEnabledMask)
        {
            return transformHandle.OnChunkBegin(in chunk, unfilteredChunkIndex, useEnabledMask, in chunkEnabledMask);
        }

        public void OnChunkEnd(in ArchetypeChunk chunk, int unfilteredChunkIndex, bool useEnabledMask, in v128 chunkEnabledMask, bool chunkWasExecuted)
        {
        }

        public void Execute([EntityIndexInChunk] int indexInChunk)
        {
            TransformAspect transform = transformHandle[indexInChunk];
            transform.TranslateWorld(transform.forwardDirection * 0.01f);
        }
    }
}
```

## EntityQuery Order Behavior Is Weird

While the above is simulation-wide deterministic, the order in which entities in
the hierarchy are updated within a thread is entirely dependent on the order of
entities within the specified `EntityQuery`. This can be fine or not fine,
depending on which parts of the transform you write, the `InheritanceFlags` of
the entities you write to, and whether or not you care about propagation
overwriting your just-written values.

If your entities use the default `InheritanceFlags`, then you can generally
write to the local transform (including stretch) and get the expected results.
However, if you write to world-space transforms, something weird happens.

Let’s suppose you have two entities named A and B, and they are the only
entities living in ChunkA and ChunkB respectively. Entity B is the sole child of
A. And currently, both are located at the origin in world-space. Let’s suppose
we have a job that sets the `worldPosition.x` to `1f`.

If ChunkA comes before ChunkB in the `EntityQuery`, then the job will first
process A, and set `worldPosition.x` to `1f`. Propagation happens immediately,
so B will inherit this change and also have `worldPosition.x` updated to `1f`.
Then, the job will process B, resulting in `worldPosition.x` being overwritten
with the value it was already set to. In the end, both entities will have moved
along the x-axis by one unit. This is what we expected.

But what if ChunkB comes before ChunkA in the `EntityQuery`? This time, the job
will process B first, and set `worldPosition.x` to `1f`. B is now offset from A,
so its `localPosition.x` is also updated to `1f`. Then, the job processes A and
sets A’s `worldPosition.x` to `1f`. The change propagates, and because B’s local
position states that B is offset from A, B’s `worldPosition.x` is updated to
`2f`.

This might be surprising. In fact, it might even seem like a race condition. It
is not, because the ordering of ChunkA and ChunkB is deterministic. If you run a
build, load the subscene, and immediately run this job, you will get the same
result every time. But how ChunkA and ChunkB are ordered is quite unpredictable,
as it is a buried implementation detail inside the Entities package. You
probably don’t want to rely on it.

Now let’s say that B used `InheritanceFlags.WorldX`. In this case, the order of
ChunkA and ChunkB stop mattering, because B ignores the propagation of A.

## Batch-Writing Arbitrary Entities Top-Down

If you do need to write transforms of arbitrary entities with predictable
hierarchy behavior, you can do so more efficiently by reasoning about the full
groups of chunks at once. This is done by defining a custom
`IJobParallelForDefer` job, where each index will process a group of chunks.
`TransformAspectParallelChunkHandle` has separate API methods for working in
this mode.

Inside the job, you can fetch each `TransformAspect`, and then fetch the
`entityInHierarchyHandle.indexInHierarchy` to get a top-down ordering for the
belonging hierarchy. And you can group entities belonging to the same hierarchy
by their roots.

When you have several entities in a hierarchy in a top-down order, you can
batch-write to them using a `Span<TransformBatchWriteCommand>`. Populate the
span by calling the static `TransformBatchWriteCommand` methods to create
command instances. Then call `ApplyTransforms()` on the span when you are done.
`ApplyTransforms()` will walk the hierarchy and apply each command while
simultaneously propagating transforms through descendants. This means there is
only one single propagation pass, making this API very efficient.

However, using this API is not for the faint-of-heart, as there is typically a
fair bit of code required to organize the entities in groups by their
hierarchies. Here’s what the example transform system looks like if it tried to
update transforms in hierarchical top-down order efficiently:

```csharp
partial struct BatchTransformSystem : ISystem
{
    EntityQuery m_query;

    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        m_query = state.Fluent().With<WorldTransform>().Build();
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var transformHandle = new TransformAspectParallelChunkHandle(SystemAPI.GetComponentLookup<WorldTransform>(false),
                                                                     SystemAPI.GetComponentTypeHandle<RootReference>(true),
                                                                     SystemAPI.GetBufferLookup<EntityInHierarchy>(true),
                                                                     SystemAPI.GetBufferLookup<EntityInHierarchyCleanup>(true),
                                                                     SystemAPI.GetEntityStorageInfoLookup(),
                                                                     ref state);

        state.Dependency = transformHandle.ScheduleChunkCaptureForQuery(m_query, state.Dependency);
        state.Dependency = transformHandle.ScheduleChunkGrouping(state.Dependency);
        state.Dependency = new BatchJob { transformHandle = transformHandle }.ScheduleParallel(transformHandle, state.Dependency);
    }

    // The job type is IJobParallelForDefer, which creates an index per group of chunks
    [BurstCompile]
    struct BatchJob : IJobParallelForDefer
    {
        public TransformAspectParallelChunkHandle transformHandle;

        HasChecker<RootReference> rootReferenceChecker;
            
        public void Execute(int index)
        {
            // If we know we are processing a single chunk of entities that are root or solo entities,
            // then we can linearly iterate through the chunk without having to concern ourselves with top-down hierarchy ordering.
            int numChunksInGroup = transformHandle.GetChunkCountForIJobParallelForDeferIndex(index);
            transformHandle.GetChunkInGroupForIJobParallelForDefer(index, 0, out var chunk, out _, out _, out _);
            if (numChunksInGroup == 1 && !rootReferenceChecker[chunk])
            {
                UpdateChunkWithOnlySoloAndRootEntitiesFast(index);
                return;
            }
            // We need to concern ourselves with top-down ordering, as multiple entities in this group might come from the same hierarchy.
            UpdateGroupInBatches(index, numChunksInGroup);
        }

        void UpdateChunkWithOnlySoloAndRootEntitiesFast(int groupIndex)
        {
            transformHandle.GetChunkInGroupForIJobParallelForDefer(groupIndex, 0, out var chunk, out var unfilteredChunkIndex, out var useEnabledMask, out var chunkEnabledMask);
            transformHandle.SetActiveChunkForIJobParallelForDefer(groupIndex, 0);

            var enumerator = new ChunkEntityEnumerator(useEnabledMask, chunkEnabledMask, chunk.Count);
            while (enumerator.NextEntityIndex(out int entityIndexInChunk))
            {
                TransformAspect transform = transformHandle[entityIndexInChunk];
                transform.TranslateWorld(transform.forwardDirection * 0.01f);
            }
        }

        void UpdateGroupInBatches(int groupIndex, int numChunksInGroup)
        {
            // Count the number of entities to process so we can stackalloc the right size.
            int numTransforms = 0;
            for (int i = 0; i < numChunksInGroup; i++)
            {
                transformHandle.GetChunkInGroupForIJobParallelForDefer(groupIndex, i, out var chunk, out var unfilteredChunkIndex, out var useEnabledMask, out var chunkEnabledMask);
                numTransforms += useEnabledMask ? math.countbits(chunkEnabledMask.ULong0) + math.countbits(chunkEnabledMask.ULong1) : chunk.Count;
            }

            Span<OrderableTransform> orderableTransforms = stackalloc OrderableTransform[numTransforms];
            numTransforms                                = 0;

            // Collect the transforms and sort them. We don't care about what order we update the independent hierarchies in,
            // just the order we process entities within each hierarchy. Therefore, we use the root Entity's Index as a key to
            // group entities in the same hierarchy together.
            for (int i = 0; i < numChunksInGroup; i++)
            {
                transformHandle.GetChunkInGroupForIJobParallelForDefer(groupIndex, i, out var chunk, out var unfilteredChunkIndex, out var useEnabledMask, out var chunkEnabledMask);
                transformHandle.SetActiveChunkForIJobParallelForDefer(groupIndex, i);

                var enumerator = new ChunkEntityEnumerator(useEnabledMask, chunkEnabledMask, chunk.Count);
                while (enumerator.NextEntityIndex(out int entityIndexInChunk))
                {
                    TransformAspect transform               = transformHandle[entityIndexInChunk];
                    var             entityInHierarchyHandle = transform.entityInHierarchyHandle;
                    orderableTransforms[numTransforms]      = new OrderableTransform
                    {
                        transformAspect  = transform,
                        indexInHierarchy = entityInHierarchyHandle.indexInHierarchy,
                        rootEntityIndex  = entityInHierarchyHandle.root.entity.Index
                    };
                    numTransforms++;
                }
            }
            orderableTransforms.Sort();

            // Create a batch for each hierarchy and update the batches
            int start = 0;
            int count = 1;
            while (start < orderableTransforms.Length)
            {
                for (; start + count < orderableTransforms.Length; count++)
                {
                    if (orderableTransforms[start].rootEntityIndex != orderableTransforms[start + count].rootEntityIndex)
                        break;
                }
                UpdateBatch(orderableTransforms.Slice(start, count));
                start = start + count;
                count = 1;
            }
        }

        void UpdateBatch(Span<OrderableTransform> batchOfTransforms)
        {
            Span<TransformBatchWriteCommand> commands = stackalloc TransformBatchWriteCommand[batchOfTransforms.Length];
            for (int i = 0; i < batchOfTransforms.Length; i++)
            {
                var transformAspect      = batchOfTransforms[i].transformAspect;
                var worldTransform       = transformAspect.worldTransform;
                worldTransform.position += worldTransform.forwardDirection * 0.01f;
                commands[i]              = TransformBatchWriteCommand.SetWorldTransform(transformAspect, worldTransform);
            }
            // ApplyTransforms() requires all commands belong to the same hierarchy, and that the commands are ordered by their target
            // transforms' indexInHierarchy values.
            commands.ApplyTransforms();
        }

        // Sort by hierarchy, then by index in hierarchy. The sorting criteria is cached as separate fields.
        struct OrderableTransform : IComparable<OrderableTransform>
        {
            public int             rootEntityIndex;
            public int             indexInHierarchy;
            public TransformAspect transformAspect;

            public int CompareTo(OrderableTransform other)
            {
                var result = rootEntityIndex.CompareTo(other.rootEntityIndex);
                if (result == 0)
                    return indexInHierarchy.CompareTo(other.indexInHierarchy);
                return result;
            }
        }
    }
}
```

This code is an example of one way to perform grouping. It is not the only way,
nor even the most optimal way. Most likely, you will want to account for other
data involved in your calculations, which may affect the optimal strategy.

## Working with the Full Hierarchies

You might have noticed by now that when you successfully obtain a
`TransformAspect`, you have been guaranteed exclusive access to the full
hierarchy. That begs the question, what else can you do with that access?

`TransformAspect` has the `entityInHierarchyHandle` property, which gives you a
way to navigate the hierarchy the entity belongs to (assuming it belongs to
one). From this, you can get `EntityInHierarchyHandle` instances of other
entities in the hierarchy. And if you pass one of these other instances into
`TransformAspect`’s indexer, you get that other entity’s `TransformAspect`. This
means you have full reign to write any transform in the hierarchy.

But why stop at transforms? After all, if you have thread-safe access to all the
entities in the hierarchy, you might want to modify some other components in the
hierarchy as well.

This is possible with `TransformsComponentLookup` and `TransformsBufferLookup`.
These are types that can be implicitly casted from `ComponentLookup` and
`BufferLookup` respectively. They enable parallel access, but require you index
them with an `EntityInHierarchyHandle` (or `Entity`, but only if the entity is a
solo entity) and a `TransformsKey`. When using `TransformAspectRootHandle`, you
can get a `TransformsKey` that is valid for all hierarchies in the chunk simply
by accessing the `transformsKey` property after calling `SetupChunk()`. As for
`TransformAspectParallelChunkHandle`, you need to call `GetTransformsKey()` and
pass in the index of the entity within the currently active chunk. This key is
only valid for the hierarchy that entity belongs to.

With this, you now have the ability to iterate entire hierarchies in parallel,
and write to any component of any entity in a hierarchy safely. Reasoning about
full hierarchies rather than individual entities can be very freeing for
gameplay logic.

## Recipes for Special Use Cases

Sometimes, you find yourself in a situation where you have data that does not
conform to the existing APIs, but you know enough assumptions that you still
want to process the data in parallel. There are a few situations where this is
safe.

### NativeArray\<Entity\> Full of Roots

If you have a `NativeArray<Entity>` that contains a set of root entities, you
can still operate on this safely in parallel. Although you have to be a little
careful to follow this pattern precisely, as safety isn’t watertight.

For this, we’ll assume that your job is of type `IJobFor` or
`IJobParallelForDefer`, scheduled with an index per entity in your
`NativeArray`. In this job, you will need a `TransformAspectRootHandle`. This
`handle` has an `entityStorageInfoLookup` property. Use this to acquire the
storage info of the entity. Next, call `handle.SetupChunk()` on the chunk
contained in the storage info. Then access the `TransformAspect` using the
storage info’s `IndexInChunk`.

For the `TransformsKey`, call
`TransformsKey.CreateFromExclusivelyAccessedRoot()` passing in the entity and
the `entityStorageInfoLookup`. Do this only after obtaining the
`TransformAspect`.

**Do not use** `handle.transformsKey`**!** That assumes you have thread-safe
access to the full ECS chunk, which you don’t have.

```csharp
struct ExampleJob : Unity.Jobs.IJobParallelForDefer
{
    [ReadOnly] public NativeArray<Entity>       rootEntities;
    public            TransformAspectRootHandle handle;
    // Other job fields go here...

    public void Execute(int index)
    {
        var entity = rootEntities[index];

        var storageInfo = handle.entityStorageInfoLookup[entity];
        handle.SetupChunk(storageInfo.Chunk);
        var transformAspect = handle[storageInfo.IndexInChunk];
        var transformsKey   = TransformsKey.CreateFromExclusivelyAccessedRoot(entity, handle.entityStorageInfoLookup);

        // Your logic goes here...
    }
}
```
