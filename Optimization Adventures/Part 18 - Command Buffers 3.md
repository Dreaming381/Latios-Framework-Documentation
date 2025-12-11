# Optimization Adventures: Part 18 – Command Buffers 3

Refactoring code to add new features is a common task amongst programmers.
There’s always something that the original code didn’t account for that makes it
unsuitable for the new task at hand. But after refactoring the code, have you
ever been surprised by a drastic performance improvement?

That just happened to me! And for a very valuable operation in the framework no
less!

In this short adventure, I’m going to provide you some insight into what
problems I was trying to solve, what I did, and what happened because of it.
There’s not going to be a whole lot of investigation, just a single profiler
result at the end. So let’s dig into how I got there.

## An API Epiphany

At the time of writing this, I’m currently in the process of completely
overhauling QVVS Transforms. The new design requires that the hierarchy is
always perfectly up-to-date at all times. That’s quite powerful, but that’s also
problematic when attempting to spawn entities through a command buffer. To set
the world-space position of the whole hierarchy, you’d need to modify every
entity in the hierarchy.

That’s not something `InstantiateCommandBuffer` can easily represent. It only
allows initializing components on the root. We’d need some kind of different
flavor, like being able to set all the buffer values or something. But then that
still wouldn’t be sufficient to instantiate an entity as a child of an already
existing entity. For that to work, something would need to inform the existing
entity’s hierarchy about the new entity right after it was created. At this
point, it seems like we’d need to add a temporary component, run a system to
perform the hierarchy update as soon as possible after the ICB was played back,
and then remove that temporary component. The temporary component is annoying.
Perhaps we could just store those values in some container instead.

Wait!

What if ICB could be populated with structs other than components? And each
struct type had a function pointer associated with it that could be called right
after playback? That would cover the QVVS use cases. But it would also be fully
extendable by users. A common request is to allow adding the newly instantiated
entity to a dynamic buffer of another entity. With this new API, a user could
define a struct with a field referencing the other entity with the dynamic
buffer. And then they could write a function pointer that when given the newly
instantiated entity, fetches and appends to that other dynamic buffer.

However, there’s one challenge with this. A user may want to make structural
changes (QVVS may even need to), which means the order entities are passed to
the function pointer must be deterministic. The current ICB implementation
doesn’t actually guarantee this. After everything gets instantiated, it shuffles
the entity/data-pointer pairs by chunk for optimal component writing. Chunk
order is non-deterministic. It didn’t really matter because each pair is
independent, and that step could even have been moved to a parallel job. But
now, we need the final order to be deterministic, and we want that after the
other components have been initialized. Therefore…

Time to redesign the ICB playback procedure.

## What Do We Need Again?

Let’s break down the requirements.

We know that for any given ICB, we want the output to be deterministic. That
means that if a parallel job recorded the same commands in separate play
sessions, the output would the same entities in the same chunk orders and the
same indices within entity queries. That also means the order we instantiate the
unique prefabs should also be deterministic. However, aside from being
deterministic, the ordering doesn’t really matter.

To make things more challenging, I figured this might be the time to work
towards future Entities package compatibility. And that means removing the
dependence on ENTITY_STORE_V1. This means, we can’t simply sort entity indices
to get determinism. Instead, we’ll have to rely solely on command order and
their `sortKeys` for deterministic behavior.

And of course, we still want this solution to be performant. That means batching
up unique commands that share the same target prefab, as well as writing the
components to the newly instantiated entities in a cache-efficient manner. It’s
a tough challenge, but like in the last adventure, I’ve learned a few tricks
since I designed ICB.

## Step by Step, Inch by Inch

The original ICB implementation (OldICB) first captured the unsorted commands
into arrays. The commands were split into two separate arrays. One array
contained prefab references and sort keys. The other contained pointers to the
component packets to assign. The latter array is actually algorithmically
calculated rather than something read from a buffer. The code looked like this:

```csharp
// Step 1: Get the prefabs and sort keys
int count              = icb.Count();
var prefabSortkeyArray = new NativeArray<PrefabSortkey>(count, Allocator.Temp, NativeArrayOptions.UninitializedMemory);
icb.m_prefabSortkeyBlockList->GetElementValues(prefabSortkeyArray);
// Step 2: Get the componentData pointers
var unsortedComponentDataPtrs = new NativeArray<UnsafeIndexedBlockList.ElementPtr>(count, Allocator.Temp, NativeArrayOptions.UninitializedMemory);
icb.m_componentDataBlockList->GetElementPtrs(unsortedComponentDataPtrs);
```

As it turns out, the same exact steps are needed for the new ICB implementation
(NewICB) as well. So this code can be copied as-is.

### Finding Unique Prefabs

The next part is where OldICB starts to break our new requirements. It performs
a 12-byte radix sort order by prefab entity and then by sort key on all the
commands. With all the commands grouped by each unique prefab entity, OldICB
then identifies each unique prefab and range of associated commands.

```csharp
// Step 3: Sort the arrays by sort key and collapse unique entities
var ranks = new NativeArray<int>(count, Allocator.Temp, NativeArrayOptions.UninitializedMemory);
RadixSort.RankSortInt3(ranks, prefabSortkeyArray);
var    sortedPrefabs           = new NativeList<Entity>(count, Allocator.Temp);
var    sortedPrefabCounts      = new NativeList<int>(count, Allocator.Temp);
var    sortedComponentDataPtrs = new NativeArray<UnsafeIndexedBlockList.ElementPtr>(count, Allocator.Temp, NativeArrayOptions.UninitializedMemory);
Entity lastEntity              = Entity.Null;
for (int i = 0; i < count; i++)
{
    var entity                 = prefabSortkeyArray[ranks[i]].prefab;
    sortedComponentDataPtrs[i] = unsortedComponentDataPtrs[ranks[i]];
    if (entity != lastEntity)
    {
        sortedPrefabs.AddNoResize(entity);
        sortedPrefabCounts.AddNoResize(1);
        lastEntity = entity;
    }
    else
    {
        ref var c = ref sortedPrefabCounts.ElementAt(sortedPrefabCounts.Length - 1);
        c++;
    }
}
```

For NewICB, we can’t include the prefab index and version values in our sort
criteria. We can only sort by the `sortKey`. Let’s start with that to at least
get our commands into a deterministic order.

```csharp
// Step 3: Sort the arrays by sort key and collapse unique entities
var ranks = new NativeArray<int>(count, Allocator.Temp, NativeArrayOptions.UninitializedMemory);
RadixSort.RankSortInt(ranks, prefabSortkeyArray);
```

However, we still need to group our commands by unique prefab. We’ll iterate the
commands and add each new unique prefab we find to a list. We’ll use a hashmap
to help map prefabs to our result list indices. And while we are add it, we can
count the number of references for each unique prefab, which will help us know
our batch sizes.

```csharp
struct UniquePrefab
{
    public Entity prefab;
    public int    start;
    public int    count;
}

var uniquePrefabs   = new UnsafeList<UniquePrefab>(count, Allocator.Temp);
var uniquePrefabMap = new UnsafeHashMap<Entity, int>(count, Allocator.Temp);
for (int i = 0; i < count; i++)
{
    var entity = prefabSortkeyArray[ranks[i]].prefab;
    if (uniquePrefabMap.TryGetValue(entity, out var uniqueIndex))
        uniquePrefabs.ElementAt(uniqueIndex).count++;
    else
    {
        uniquePrefabMap.Add(entity, uniquePrefabs.Length);
        uniquePrefabs.AddNoResize(new UniquePrefab { prefab = entity, count = 1 });
    }
}
```

Now that we have our unique prefabs, we can prefix-sum them so that we have
allocated ranges.

```csharp
int running = 0;
for (int i = 0; i < uniquePrefabs.Length; i++)
{
    ref var u  = ref uniquePrefabs.ElementAt(i);
    u.start    = running;
    running   += u.count;
    u.count    = 0;
}
```

You’ll notice that we reset the count back to 0. This is because we will count
it back up as we iterate commands and assign the data pointers in the next step.

```csharp
var sortedComponentDataPtrs = new NativeArray<UnsafeIndexedBlockList.ElementPtr>(count, Allocator.Temp, NativeArrayOptions.UninitializedMemory);
for (int i = 0; i < count; i++)
{
    ref var u                                  = ref uniquePrefabs.ElementAt(uniquePrefabMap[prefabSortkeyArray[ranks[i]].prefab]);
    sortedComponentDataPtrs[u.start + u.count] = unsortedComponentDataPtrs[ranks[i]];
    u.count++;
}
```

We now have sorted the data pointers to be deterministically grouped by unique
prefabs, which we also have in deterministic order. At this point, we no longer
need `uniquePrefabMap`, `ranks`, nor `prefabSortKeyArray`. Those can all be
allowed to evict from cache.

Let’s stop to compare the two solutions so far. In NewICB, we use a smaller
radix sort, which is faster. However, we then perform two passes over the data
using a hashmap, versus the single combining pass of OldICB. Fortunately, the
number of unique prefabs is often relatively small at scale, so we won’t be
jumping around in memory much. I should also note here that even though we often
indirectly index things through the ranks array, this still tends to be fairly
cache-efficient as the ranks have lots of consecutive runs in them (an ECS chunk
will write multiple commands out in a row).

Overall, it isn’t very clear yet whether OldICB or NewICB is the winner. It
comes down to how expensive the extra 8 passes of radix sort are compared to two
hashmap passes.

### Instantiating Entities

Next, both steps need to instantiate the prefabs. This process is nearly
identical, with the only differences being how the unique prefabs and counts
have been stored. Here’s OldICB:

```csharp
// Step 4: Instantiate the prefabs
var instantiatedEntities = new NativeArray<Entity>(count, Allocator.Temp, NativeArrayOptions.UninitializedMemory);
var typesWithDataToAdd   = BuildComponentTypesFromFixedList(icb.m_state->typesWithData);
int startIndex           = 0;
for (int i = 0; i < sortedPrefabs.Length; i++)
{
    var firstEntity = em.Instantiate(sortedPrefabs[i]);
    em.AddComponent(firstEntity, typesWithDataToAdd);
    em.AddComponent(firstEntity, icb.m_state->tagsToAdd);
    instantiatedEntities[startIndex] = firstEntity;
    startIndex++;

    if (sortedPrefabCounts[i] - 1 > 0)
    {
        var subArray = instantiatedEntities.GetSubArray(startIndex, sortedPrefabCounts[i] - 1);
        em.Instantiate(firstEntity, subArray);
        startIndex += subArray.Length;
    }
}
```

And this is what NewICB looks like:

```csharp
// Step 4: Instantiate the prefabs
var instantiatedEntities = new NativeArray<Entity>(count, Allocator.Temp, NativeArrayOptions.UninitializedMemory);
var typesWithDataToAdd   = BuildComponentTypesFromFixedList(icb.m_state->typesWithData);
int startIndex           = 0;
for (int i = 0; i < uniquePrefabs.Length; i++)
{
    var uniquePrefab = uniquePrefabs[i];
    var firstEntity  = em.Instantiate(uniquePrefab.prefab);
    em.AddComponent(firstEntity, typesWithDataToAdd);
    em.AddComponent(firstEntity, icb.m_state->tagsToAdd);
    instantiatedEntities[startIndex] = firstEntity;
    startIndex++;

    if (uniquePrefab.count - 1 > 0)
    {
        var subArray = instantiatedEntities.GetSubArray(startIndex, uniquePrefab.count - 1);
        em.Instantiate(firstEntity, subArray);
        startIndex += subArray.Length;
    }
}
```

Yeah. They really aren’t much different. But the next part is where things get
really wild.

## Doing Things the Old-Fashioned Way

Next, OldICB fetches the storage info of each instantiated entity. Then it
performs another reordering of the instantiated entities and associated data
pointers. Once those are reordered, it batches them up by chunk.

```csharp
// Step 5: Get locations of new entities
var locations = new NativeArray<EntityStorageInfo>(count, Allocator.Temp);
for (int i = 0; i < count; i++)
{
    locations[i] = em.GetStorageInfo(instantiatedEntities[i]);
}
// Step 6: Sort chunks and build final lists
RadixSort.RankSortInt3(ranks, locations.Reinterpret<WrappedEntityLocationInChunk>());
chunks.Capacity      = count;
chunkRanges.Capacity = count;
indicesInChunks.ResizeUninitialized(count);
componentDataPtrs.ResizeUninitialized(count);
ArchetypeChunk lastChunk = default;
for (int i = 0; i < count; i++)
{
    var loc              = locations[ranks[i]];
    indicesInChunks[i]   = loc.IndexInChunk;
    componentDataPtrs[i] = sortedComponentDataPtrs[ranks[i]];
    if (loc.Chunk != lastChunk)
    {
        chunks.AddNoResize(loc.Chunk);
        chunkRanges.AddNoResize(new int2(i, 1));
        lastChunk = loc.Chunk;
    }
    else
    {
        ref var c = ref chunkRanges.ElementAt(chunkRanges.Length - 1);
        c.y++;
    }
}
```

There’s some variables here that seem to show up out of nowhere. OldICB actually
runs all of its logic inside jobs. However, the jobs are manually executed on
the main thread in modern versions of the framework. `chunks`, `chunkRanges`,
`indicesInChunks`, and `componentDataPtrs` are all outputs of this first job.
The second job iterates chunks and assigns the components using
`UnsafeUtility.MemCpy()`.

```csharp
private struct WriteComponentDataJob
{
    [ReadOnly] public InstantiateCommandBufferUntyped                icb;
    [ReadOnly] public NativeArray<ArchetypeChunk>                    chunks;
    [ReadOnly] public NativeArray<int2>                              chunkRanges;
    [ReadOnly] public NativeArray<int>                               indicesInChunks;
    [ReadOnly] public NativeArray<UnsafeIndexedBlockList.ElementPtr> componentDataPtrs;
    [ReadOnly] public EntityTypeHandle                               entityHandle;
    public DynamicComponentTypeHandle                                t0;
    public DynamicComponentTypeHandle                                t1;
    public DynamicComponentTypeHandle                                t2;
    public DynamicComponentTypeHandle                                t3;
    public DynamicComponentTypeHandle                                t4;

    public void Execute(int i)
    {
        var chunk   = chunks[i];
        var range   = chunkRanges[i];
        var indices = indicesInChunks.GetSubArray(range.x, range.y);
        var ptrs    = componentDataPtrs.GetSubArray(range.x, range.y);
        switch (icb.m_state->typesSizes.Length)
        {
            case 1: DoT0(chunk, indices, ptrs); return;
            case 2: DoT1(chunk, indices, ptrs); return;
            case 3: DoT2(chunk, indices, ptrs); return;
            case 4: DoT3(chunk, indices, ptrs); return;
            case 5: DoT4(chunk, indices, ptrs); return;
        }
    }

    void DoT0(ArchetypeChunk chunk, NativeArray<int> indices, NativeArray<UnsafeIndexedBlockList.ElementPtr> dataPtrs)
    {
        var   entities = chunk.GetNativeArray(entityHandle);
        var   t0Size   = icb.m_state->typesSizes[0];
        var   t0Array  = chunk.GetDynamicComponentDataArrayReinterpret<byte>(ref t0, t0Size);
        byte* t0Ptr    = (byte*)t0Array.GetUnsafePtr();
        for (int i = 0; i < indices.Length; i++)
        {
            var index   = indices[i];
            var dataPtr = dataPtrs[i].ptr;
            UnsafeUtility.MemCpy(t0Ptr + index * t0Size, dataPtr, t0Size);
        }
    }

    void DoT1(ArchetypeChunk chunk, NativeArray<int> indices, NativeArray<UnsafeIndexedBlockList.ElementPtr> dataPtrs)
    {
        var   entities = chunk.GetNativeArray(entityHandle);
        var   t0Size   = icb.m_state->typesSizes[0];
        var   t1Size   = icb.m_state->typesSizes[1];
        var   t0Array  = chunk.GetDynamicComponentDataArrayReinterpret<byte>(ref t0, t0Size);
        var   t1Array  = chunk.GetDynamicComponentDataArrayReinterpret<byte>(ref t1, t1Size);
        byte* t0Ptr    = (byte*)t0Array.GetUnsafePtr();
        byte* t1Ptr    = (byte*)t1Array.GetUnsafePtr();
        for (int i = 0; i < indices.Length; i++)
        {
            var index   = indices[i];
            var dataPtr = dataPtrs[i].ptr;
            UnsafeUtility.MemCpy(t0Ptr + index * t0Size, dataPtr, t0Size);
            dataPtr += t0Size;
            UnsafeUtility.MemCpy(t1Ptr + index * t1Size, dataPtr, t1Size);
        }
    }

    // ...
```

I honestly have no idea why OldICB was fetching the `entities` array. It doesn’t
use it for anything.

## The Shortcut

The second reordering by chunk is a problem, because the order of chunks is not
deterministic. And while that doesn’t matter for assigning components, it does
matter for passing the instantiated entities and custom commands to a
user-written function pointer. Is there a way to get chunk batching based on
instantiation order?

Actually, yeah. Kinda.

Let’s think through what actually happens when we instantiate an entity. All we
are doing is creating new entities. No existing entities ever have to move
around. And when we instantiate multiple copies of the same entity, all the
entities have the same archetype, which means they can all go into the same
chunks together. This means that the first set of entities will all be in the
same chunk up until either they all fit or the chunk is full. Then whatever is
left over will go into the next chunk, and so-on. There’s already chunk-level
batching happening for each entity.

The only benefit OldICB has is if multiple unique prefabs had all their entities
fit within the same chunk. However, at scale, a chunk can’t fit that many
entities, so this is quite unlikely.

For new ICB, instead of a switch case evaluated per chunk, I decided to use an
interface for writing a given unique prefab’s new entities within a chunk, and
then use a generic algorithm for walking chunks by unique prefab.

Here’s an example of one of the interface implementations:

```csharp
struct ChunkExecuteT1 : IChunkProcessor
{
    public DynamicComponentTypeHandle t0;
    public DynamicComponentTypeHandle t1;
    public int                        t0Size;
    public int                        t1Size;

    public void Execute(in ArchetypeChunk chunk, NativeArray<UnsafeIndexedBlockList.ElementPtr> componentDataPtrs, int chunkStart)
    {
        var t0Ptr  = (byte*)chunk.GetDynamicComponentDataArrayReinterpret<byte>(ref t0, t0Size).GetUnsafePtr();
        var t1Ptr  = (byte*)chunk.GetDynamicComponentDataArrayReinterpret<byte>(ref t1, t1Size).GetUnsafePtr();
        t0Ptr     += chunkStart * t0Size;
        t1Ptr     += chunkStart * t1Size;
        for (int i = 0; i < componentDataPtrs.Length; i++)
        {
            UnsafeUtility.MemCpy(t0Ptr + i * t0Size, componentDataPtrs[i].ptr,          t0Size);
            UnsafeUtility.MemCpy(t1Ptr + i * t1Size, componentDataPtrs[i].ptr + t0Size, t1Size);
        }
    }
}
```

It is worth noting that these processors assume all the instantiated entities
are consecutive within the chunk. OldICB didn’t make that assumption, which was
a bit of a missed opportunity.

And here’s the walking algorithm:

```csharp
static void ProcessEntitiesInChunks<T>(EntityManager em, UnsafeList<UniquePrefab> uniquePrefabs, NativeArray<Entity> entities,
                                        NativeArray<UnsafeIndexedBlockList.ElementPtr> componentDataPtrs, ref T processor) where T : unmanaged, IChunkProcessor
{
    int offset = 0;
    for (int uniquePrefabIndex = 0; uniquePrefabIndex < uniquePrefabs.Length; uniquePrefabIndex++)
    {
        int countRemaining = uniquePrefabs[uniquePrefabIndex].count;
        while (countRemaining > 0)
        {
            var info           = em.GetStorageInfo(entities[offset]);
            var countToProcess = math.min(countRemaining, info.Chunk.Count - info.IndexInChunk);
            var subArray       = componentDataPtrs.GetSubArray(offset, countToProcess);
            processor.Execute(in info.Chunk, subArray, info.IndexInChunk);
            offset         += countToProcess;
            countRemaining -= countToProcess;
        }
    }
}
```

There’s something really interesting happening here. For each unique prefab, we
get the location of the first entity. And then we just assume that the following
entities come from the same unique prefab, and are in the same order as our
instantiated `entities` array. This means we don’t need to look up the storage
info of all the remaining entities, which last time we discovered was expensive
(they are random accesses). It is only when we either exhaust the unique
prefab’s new entities, or exhaust the chunk’s capacity that we look up another
entity location.

With that, the final step in NewICB is a switch case block to set up the
appropriate processor and then call the walker method above. That’s it!

## The Surprise

We are using a hashmap which is slower. But we are also doing much less radix
sorting passes which is faster. But does any of this matter for performance?
After all, there’s the very heavy operation right in the middle, which is
actually instantiating the entities. And we haven’t changed any aspect of that
at all.

At the very least, I didn’t want to introduce any major performance regressions,
so I decided to benchmark LSSS Sector 2 Mission 6 (same as last time). I saved
two profiler captures, and fed each of them into the Profiler Analyzer.

Surprisingly, I don’t think I’ve shown off the Profiler Analyzer in any of these
adventures so far. It is a useful tool for getting a big picture view of what is
happening across multiple frames. I was expecting the two profilers to be
statistically similar, which would be enough confirmation for me to greenlight
the new algorithm. Instead, I saw this:

![A screenshot of a computer AI-generated content may be
incorrect.](media/eda22a08b1873b1f44e5aad5713272fc.png)

Left is OldICB, and right is NewICB.

I’ve highlighted the most prominent pair of ICBs over many frames, which is
firing bullets. Ships fire bullets pretty much all the time, though especially
when they all spawn. As you can see, NewICB did much better, both at the spikes,
but also just all around in general.

For ship spawning, the advantage wasn’t as clear. But `LinkedEntityGroup` is
involved, so that makes the instantiation part a bigger factor to the overall
time.

![A screenshot of a computer AI-generated content may be
incorrect.](media/b48747e680b3a5dce2f34537ae1277cf.png)

But even spawning explosions shows a clear trend (explosions have LEGs too):

![A screenshot of a graph AI-generated content may be
incorrect.](media/5fdcc9ce1524d31fd98b2b56d214e57c.png)

NewICB is provably faster, and especially for simpler entities. In the case of
bullets, it is nearly twice as fast!

## What’s Next?

Turns out, that a little better understanding of the instantiation process can
make a big difference. Though because I discovered this speed up by accident, I
couldn’t tell you which detail specifically had the biggest impact.

The takeaway here is that as you gain experience, your intuition and ability to
spot simplifications will also improve. And that can have a real beneficial
impact on performance. Just make sure you profile.

Anyways, thanks for reading this little surprise adventure. I suspect the next
one will be either related to the new QVVS design, or some crazy pixel
processing of text rendering. But who knows? I certainly don’t.

## Try It Yourself

Both OldICB and NewICB can be found in this version of LSSS:

<https://github.com/Dreaming381/lsss-wip/tree/044567336114892ceb4372fa243c5e31d6423711>
