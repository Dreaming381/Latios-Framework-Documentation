# Getting Started with Psyshock Physics – Part 4

In the previous part, we learned about FindPairs and how to perform fast
pair-wise searches. In this part, we’ll explore how to cache the results of
these pairs to perform successive operations using `PairStream` and ForEachPair.

## The Use Case

Unlike the previous APIs discussed which are able to generally leverage common
knowledge, `PairStream` is a unique invention to Psyshock. Its closest relative
is `NativeStream`, which is already a not commonly understood collection, and
`PairStream` is different from `NativeStream` in a lot of ways.

To help build an intuition, we will explore these APIs through an example. The
most common use case for `PairStream` is to store constraint solver states in a
simulation. We’ll discuss that in a future part, but for this introduction,
we’ll use a gameplay mechanic that happens to be well-suited for `PairStream` to
solve.

We start with **zones**. These are simply colliders with some zone components.
We also have **players**. The game is simple, when a player is in a zone, the
player should receive points for that frame. This is simple enough to do with
just FindPairs:

```csharp
struct PlayerPoints : IComponentData
{
    public float points;
}

struct ZonePoints : IComponentData
{
    public float pointsPerFrame;
}

partial struct ZonePointsSystem : ISystem
{
    BuildCollisionLayerTypeHandles handles;
    EntityQuery                    playerQuery;
    EntityQuery                    zoneQuery;

    public void OnCreate(ref SystemState state)
    {
        handles     = new BuildCollisionLayerTypeHandles(ref state);
        playerQuery = state.Fluent().With<PlayerPoints>().PatchQueryForBuildingCollisionLayer().Build();
        zoneQuery   = state.Fluent().With<ZonePoints>().PatchQueryForBuildingCollisionLayer().Build();
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var buildPlayersJh = Physics.BuildCollisionLayer(playerQuery, handles).ScheduleParallel(out var playerLayer, state.WorldUpdateAllocator, state.Dependency);
        var buildZonesJh   = Physics.BuildCollisionLayer(zoneQuery, handles).ScheduleParallel(out var zoneLayer, state.WorldUpdateAllocator, state.Dependency);

        var findPairsProcessor = new PlayerVsZoneFindPairsProcessor
        {
            playerPointsLookup = SystemAPI.GetComponentLookup<PlayerPoints>(false),
            zonePointsLookup   = SystemAPI.GetComponentLookup<ZonePoints>(true)
        };

        state.Dependency = Physics.FindPairs(playerLayer, zoneLayer, findPairsProcessor).ScheduleParallel(JobHandle.CombineDependencies(buildPlayersJh, buildZonesJh));
    }

    struct PlayerVsZoneFindPairsProcessor : IFindPairsProcessor
    {
        public PhysicsComponentLookup<PlayerPoints>   playerPointsLookup;
        [ReadOnly] public ComponentLookup<ZonePoints> zonePointsLookup;

        public void Execute(in FindPairsResult result)
        {
            if (Physics.DistanceBetween(result.colliderA, result.transformA, result.colliderB, result.transformB, 0f, out _))
            {
                playerPointsLookup.GetRW(result.entityA).ValueRW.points += zonePointsLookup[result.entityB].pointsPerFrame;
            }
        }
    }
}
```

However, there’s a catch. When multiple players are in a zone, the points the
zone awards in that frame needs to be distributed between the players. The more
players are in one zone, the less points each receives.

Now we have a dilemma. A zone won’t know how many players are in a zone and how
many points it should give out until after the FindPairs operation completes.
But then it still needs to distribute the points. After figuring out our player
totals per zone, we’d ideally prefer to loop over all our pairs again to assign
the points.

*Q: Couldn’t the zone just keep a list of players in a DynamicBuffer?*

*A: Let’s assume zones can overlap so that a player can be in multiple zones at
once. In that case, we wouldn’t be able to add points to the player safely in
parallel.*

## Our First PairStream

To solve our dilemma, we will create a `PairStream`, and write all our pairs to
it. `PairStream` has several constructors, all of which are simply different
ways to provide it with the number of buckets the `CollisionLayers` use. We can
pass a `CollisionLayer` directly (even if it is being used in a job), or we can
pass its settings, or even some raw settings values. Similar to other
containers, we can call `AsParallelWriter()` to get something we can write to in
a parallel FindPairs operation.

```csharp
var pairStream = new PairStream(playerLayer, state.WorldUpdateAllocator);

var findPairsProcessor = new PlayerVsZoneFindPairsProcessor
{
    pairStream = pairStream.AsParallelWriter()
};
```

In our FindPairs processor, we accumulate a count of players in each zone in a
component, so that by the end, it will represent our total. And then we write
out our pair to the `PairStream`. For this first example, we’ll use
`AddPairRaw()`. The first argument is a key we can get from our
`FindPairsResult`, and it contains all the info about the pair’s origins. Next,
we can specify for each of the pair’s entities whether or not we will want write
access later. Not requesting read-write access for an entity can help increase
parallelism if the entity is used in a lot of pairs. In our case, we will need
to write the points to the player, but we will only need to read the zone data
to calculate the points. We’ll default/ignore all other arguments.

```csharp
struct ZonePlayerCount : IComponentData
{
    public int playerCount;
}

struct PlayerVsZoneFindPairsProcessor : IFindPairsProcessor
{
    public PairStream.ParallelWriter               pairStream;
    public PhysicsComponentLookup<ZonePlayerCount> zonePlayerCountLookup;

    public void Execute(in FindPairsResult result)
    {
        if (Physics.DistanceBetween(result.colliderA, result.transformA, result.colliderB, result.transformB, 0f, out _))
        {
            zonePlayerCountLookup.GetRW(result.entityA).ValueRW.playerCount++;
            pairStream.AddPairRaw(result.pairStreamKey, true, false, default, default, out _);
        }
    }
}
```

Next, we need to process our cached pairs. For this, we use an
`IForEachPairProcessor`. The similarities to an `IFindPairsProcessor` is
hopefully apparent.

```csharp
struct SecondPassForEachPairProcessor : IForEachPairProcessor
{
    public PhysicsComponentLookup<PlayerPoints>        playerPointsLookup;
    [ReadOnly] public ComponentLookup<ZonePlayerCount> zonePlayerCountLookup;
    [ReadOnly] public ComponentLookup<ZonePoints>      zonePointsLookup;

    public void Execute(ref PairStream.Pair pair)
    {
        var pointsInZone                                       = zonePointsLookup[pair.entityB].pointsPerFrame;
        var playerCountInZone                                  = zonePlayerCountLookup[pair.entityB].playerCount;
        playerPointsLookup.GetRW(pair.entityA).ValueRW.points += pointsInZone / playerCountInZone;
    }
}
```

Scheduling is also straightforward:

```csharp
var forEachPairProcessor = new SecondPassForEachPairProcessor
{
    playerPointsLookup = SystemAPI.GetComponentLookup<PlayerPoints>(false),
    zonePlayerCountLookup = SystemAPI.GetComponentLookup<ZonePlayerCount>(true),
    zonePointsLookup = SystemAPI.GetComponentLookup<ZonePoints>(true),
};

state.Dependency = Physics.ForEachPair(pairStream, forEachPairProcessor).ScheduleParallel(state.Dependency);
```

Similar to FindPairs, there are multiple scheduling options for scheduling a
ForEachPair operation. And they generally reflect the same behaviors of
FindPairs. However, `ScheduleParallel()` has a little extra trick to drastically
increase parallelism if possible. Similar to FindPairs, ForEachPair will use two
phases. But unlike FindPairs, ForEachPair knows all the second phase pairs
already during the first phase. Using this knowledge, ForEachPair will allocate
one extra thread during the first phase to further organize the pairs in the
second phase into independent clusters. An entity that was marked with
read-write access is only ever allowed in a single cluster, while an entity that
was marked as read-only can be used in multiple clusters. In our scenario, this
means each player entity lives in its own cluster, and will have one or more
pairs for different zones it is within.

## Storing Information in Pairs

PairStream has a lot more capabilities than just remembering entity pairs. It
can remember additional data associated with each pair.

To illustrate this, let’s suppose that instead of dividing the points in a zone
evenly amongst the players within, we instead want to distribute the points
weighted by how close each player is to the center. Instead of totaling player
counts, the zone will total inverse distances to each player. This means that we
will need to calculate the weight in FindPairs, but we will need that weight
again in ForEachPair. We could recalculate it again, but there’s a catch.
ForEachPair doesn’t actually remember the colliders and transforms we used in
FindPairs. We’d have to read that data from `ComponentLookups`.

The better option is to leverage `PairStream`’s internal stream storage and save
the computed weights. To do this, when adding our pairs, we will instead use
`AddPairAndGetRef<T>()`. This method will allocate an instance of type `T`
within the stream and return it to us by `ref` so we can initialize it.

```csharp
struct PlayerVsZoneFindPairsProcessor : IFindPairsProcessor
{
    public PairStream.ParallelWriter                     pairStream;
    public PhysicsComponentLookup<ZonePlayerTotalWeight> zonePlayerTotalWeightLookup;

    public void Execute(in FindPairsResult result)
    {
        if (Physics.DistanceBetween(result.colliderA, result.transformA, result.colliderB, result.transformB, 0f, out _))
        {
            ref var weight = ref pairStream.AddPairAndGetRef<float>(result.pairStreamKey, true, false, out _);

            weight = 1f / math.max(0.001f, math.distance(result.transformA.position, result.transformB.position));
            zonePlayerTotalWeightLookup.GetRW(result.entityA).ValueRW.totalWeight += weight;
        }
    }
}
```

Now with our weight stored in the pair, we can read it back by using
`Pair.GetRef<T>()`. We have to make sure we pass in the same type. If we don’t,
a safety check will throw an exception when safety checks are enabled.
(`PairStream` stores the `BurstRuntime` 32-bit type hash to validate this.)

```csharp
struct PlayerVsZoneFindPairsProcessor : IFindPairsProcessor
{
    public PairStream.ParallelWriter                     pairStream;
    public PhysicsComponentLookup<ZonePlayerTotalWeight> zonePlayerTotalWeightLookup;

    public void Execute(in FindPairsResult result)
    {
        if (Physics.DistanceBetween(result.colliderA, result.transformA, result.colliderB, result.transformB, 0f, out _))
        {
            ref var weight = ref pairStream.AddPairAndGetRef<float>(result.pairStreamKey, true, false, out _);

            weight = 1f / math.max(0.001f, math.distance(result.transformA.position, result.transformB.position));
            zonePlayerTotalWeightLookup.GetRW(result.entityA).ValueRW.totalWeight += weight;
        }
    }
}
```

Something interesting you may notice is that we can get the stored data by ref.
This is thread-safe writable. We can modify it for future ForEachPair
operations, or we can completely replace it with a totally different type by
calling `Pair.ReplaceRef<T>()`. As for the “raw” APIs, these are useful if you’d
instead to prefer with raw untyped allocations instead. Type validation will be
disabled if you do this.

Also, it is worth noting that even though there is an API to replace data, the
old data will still live in the `PairStream`. It simply won’t be directly
referenced by the pair anymore. This characteristic may prove practical at
times, but it is also something to be wary of for long-persisting `PairStream`
instances.

## Dynamic Stream Allocations

We can only associate a single allocated instance to a pair in our PairStream.
But what happens if we need dynamic data? Are we stuck with using the raw API?

Not at all!

Let’s suppose that some of our zones have multiple focal points, rather than one
at the center. These focal points have bias factors that dictate how much of the
zone’s points should be accredited to their influence. And players are evaluated
against all focal points.

Now, we need to sometimes store inverse distances for each focal point within a
pair. When we add a pair, we get back the Pair instance as an out parameter. We
can call `Pair.Allocate<T>()` to allocate a `StreamSpan<T>` which we can store
in a custom struct in our root pair object. In our case, we can also just use
this directly as our root type.

`StreamSpan` can be converted into a `System.Span`, but you can also index it
directly. It is effectively a nested array inside a `PairStream`, and you should
only ever use it when creating a pair or within a ForEachPair operation. Just
like the root data stored in the pair, `StreamSpan`’s are also safe to write or
replace in subsequent ForEachPair operations. And you can even nest them.

We still have one more problem. Some of our zones still need the old
implementation without focal points. When we write a pair, it would be nice if
we can also store some hint as to whether we allocated a `StreamSpan<float>` or
just a single `float`. For this, we can use some other properties of the `Pair`:
`userByte` and `userShort`. These properties exist independent of the type we
store, allowing us to store additional metadata for the pair to help us later
on. Here, we’ll use `userByte` and store a 0 for the center weighting, and 1 for
focal weighting.

This is what our two processors now look like:

```csharp
struct ZoneFocal : IBufferElementData
{
    public float3 relativeLocation;
    public float bias;
    public float totalWeight;
}

struct PlayerVsZoneFindPairsProcessor : IFindPairsProcessor
{
    public PairStream.ParallelWriter                     pairStream;
    public PhysicsComponentLookup<ZonePlayerTotalWeight> zonePlayerTotalWeightLookup;
    public PhysicsBufferLookup<ZoneFocal>                zoneFocalLookup;

    public void Execute(in FindPairsResult result)
    {
        if (Physics.DistanceBetween(result.colliderA, result.transformA, result.colliderB, result.transformB, 0f, out _))
        {
            if (zoneFocalLookup.TryGetComponent(result.entityB, out var focals))
            {
                ref var weightSpan = ref pairStream.AddPairAndGetRef<StreamSpan<float> >(result.pairStreamKey, true, false, out var pair);

                weightSpan = pair.Allocate<float>(focals.Length, NativeArrayOptions.UninitializedMemory);

                pair.userByte = 1;

                for (int i = 0; i < weightSpan.length; i++)
                {
                    var focal             = focals[i];
                    var transformedFocal  = qvvs.TransformPoint(result.transformB, focal.relativeLocation);
                    weightSpan[i]         = 1f / math.max(0.001f, math.distance(result.transformA.position, transformedFocal));
                    focal.totalWeight    += weightSpan[i];
                    focals[i]             = focal;
                }
            }
            else
            {
                ref var weight = ref pairStream.AddPairAndGetRef<float>(result.pairStreamKey, true, false, out var pair);

                pair.userByte = 0;

                weight = 1f / math.max(0.001f, math.distance(result.transformA.position, result.transformB.position));

                zonePlayerTotalWeightLookup.GetRW(result.entityA).ValueRW.totalWeight += weight;
            }
        }
    }
}

struct SecondPassForEachPairProcessor : IForEachPairProcessor
{
    public PhysicsComponentLookup<PlayerPoints>              playerPointsLookup;
    [ReadOnly] public ComponentLookup<ZonePlayerTotalWeight> zonePlayerTotalWeightLookup;
    [ReadOnly] public ComponentLookup<ZonePoints>            zonePointsLookup;
    [ReadOnly] public BufferLookup<ZoneFocal>                zoneFocalLookup;

    public void Execute(ref PairStream.Pair pair)
    {
        var pointsInZone = zonePointsLookup[pair.entityB].pointsPerFrame;

        if (pair.userByte == 0)
        {
            var totalWeight = zonePlayerTotalWeightLookup[pair.entityB].totalWeight;

            ref var playerWeight = ref pair.GetRef<float>();

            playerPointsLookup.GetRW(pair.entityA).ValueRW.points += pointsInZone * playerWeight / totalWeight;
        }
        else if (pair.userByte == 1)
        {
            var     focals        = zoneFocalLookup[pair.entityB];
            var     playerWeights = pair.GetRef<StreamSpan<float> >();
            ref var playerPoints  = ref playerPointsLookup.GetRW(pair.entityA).ValueRW;

            for (int i = 0; i < focals.Length; i++)
            {
                playerPoints.points += focals[i].bias * pointsInZone * playerWeights[i] / focals[i].totalWeight;
            }
        }
    }
}
```

## Extra Properties and Features

Hopefully this example has provided a foundational understanding of what
`PairStream` is and how it can be used. There are several more things to be
aware of.

For example, the `Pair` can additionally provide whether each entity in the pair
was granted write-access. Additionally, a `Pair` provides the `streamIndex` it
belongs to at present. The `streamIndex` can be used for writing to command
buffers, similar to the `jobIndex` in `FindPairsResult`. However, it has little
other correlation, and pairs are allowed to be moved between streams between
ForEachPair operations.

Another notable API is when you need to add pairs manually to a `PairStream`.
This is only supported with a normal `PairStream`, and not a `ParallelWriter`.
You will need to know the bucket indices of the entities you wish to add. To
obtain this, you can use a `CollisionLayerBucketIndexCalculator` to compute the
bucket for any `Aabb`. A common use case for this would be registering joint
pairs in a constraint solver.

There are also APIs to add pairs using passed-in `Pair` instances. The primary
use case for this is to forward a pair you are currently processing in a
ForEachPair operation into a new `PairStream`. However, if you simply wish to
filter pairs, you can instead choose to disable them. Each pair has an enabled
state, and disabled pairs are excluded from the ForEachPair operation by
default. Although you can specify to include the disabled pairs at schedule
time.

If two `PairStreams` were created with the same configuration and same
allocator, you can migrate the internal memory of one `PairStream` to another,
effectively concatenating them. The `PairStream` whose memory was stolen from
will be left in an empty state ready for new pairs to be added if you desire.
Migrated pairs will be in a deterministic but not necessarily original order.

Lastly, `PairStream` has an `Enumerator` type if you just want to iterate over
all pairs in a `foreach` block. Disabled pairs are included in this enumeration.

## Up Next

With PairStream, you now have a powerful tool to create complex spatial trigger
event processing pipelines that can be executed fully multi-threaded. But as
said at the beginning, this feature was mainly designed for physics simulation
solvers. Which means we need to start exploring simulation. In the next part,
we’ll cover basic particle and rigid body motion, including the treacherous
details of rigid body rotation.

Continue to [Part 5](Getting%20Started%20-%20Part%205.md).
