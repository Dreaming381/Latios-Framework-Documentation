# Getting Started with Psyshock Physics – Part 3

In the previous part, we learned about collision layers and how we could query
arbitrary things against them. This time, we’ll look at FindPairs, another tool
designed to leverage everything the collision layer has to offer.

## IFindPairsProcessor and Scheduling FindPairs

Unlike `FindObjects` which allows you to search a single AABB region within a
`CollisionLayer`, FindPairs allows you to search the AABB of each element in the
`CollisionLayer` against all the other elements in the `CollisionLayer` in a
highly-optimized parallel operation. Each time FindPairs finds two elements
whose AABBs overlap, it must give the results back to you.

Similar to how an `IJobEntity` reports each filtered entity via the `Execute()`
method, FindPairs reports each found pair via an `Execute()` method in your own
provided `IFindPairsProcessor`. You can think of an `IFindPairsProcessor` like a
job struct. It is an unmanaged struct usually with some native container fields,
and when running in parallel, there’s an instance of the struct for each thread.
The only real difference is that you don’t need the `[BurstCompile]` attribute.
The attribute is assumed.

Here's a basic `IFindPairsProcessor` that logs each pair found:

```csharp
struct LogPairsProcessor : IFindPairsProcessor
{
    public void Execute(in FindPairsResult result)
    {
        UnityEngine.Debug.Log($"Found pair: {result.bodyA.entity.ToFixedString()}, {result.bodyB.entity.ToFixedString()}");
    }
}
```

You’ll see the `Execute()` method provides a `FindPairsResult`. This struct
contains all sorts of useful properties, like the `ColliderBody` instances for
each element in the pair.

To schedule this in an `ISystem`, you simply need this line:

```csharp
state.Dependency = Physics.FindPairs(ourCollisionLayer, new LogPairsProcessor()).ScheduleParallel(state.Dependency);
```

Just like when building a collision layer, FindPairs also uses a fluent API.
FindPairs has five scheduling modes:

-   .RunImmediate – Run without scheduling a job. You can call this from inside
    a job.
-   .Run – Run on the main thread with Burst.
-   .ScheduleSingle – Run on a worker thread with Burst.
-   .ScheduleParallel – Run on multiple worker threads with Burst.
-   .ScheduleParallelUnsafe – Run with maximum parallelism, but disable some
    features

## Parallel Write Access

FindPairs has a superpower. When scheduling using `ScheduleParallel()`, the two
entities provided in the `FindPairsResult` are deterministically thread-safe
writeable. That means you can safely read and write to any component or buffer
on either entity in the pair without any kind of race condition or concern about
order of operations. The only rule for this is that an entity can only be
referenced by a single `ColliderBody` in a `CollisionLayer`.

FindPairs uses the buckets in the `CollisionLayer` to parallelize the work.
Thus, more buckets often mean more opportunities for parallelism. But a
heavily-populated cross-bucket can interfere with that. That’s because FindPairs
dispatches in two phases, first for each of the buckets, and then again to test
the cross-bucket against all the cell buckets. The second phase is
single-threaded (dual-threaded in the editor as an extra thread is dispatched to
validate no entity is repeated in the `CollisionLayer`). You can see these
phases in the timeline view of the profiler.

So yeah, FindPairs has awesome thread-safety guarantees. But if you try to write
to the components of the entity pairs using `ComponentLookup`, you will still
get safety errors. That’s because Unity’s job system doesn’t know this is safe.
You could get around this with the `[NativeDisableParallelForRestriction]`
attribute, but Psyshock provides a safer alternative. In your
`IFindPairsProcessor`, simply replace your `ComponentLookup` with
`PhysicsComponentLookup`.

Here's an example where two ship entities take damage when they collide.
FindPairs reports pairs where the AABBs are overlapping, so the
`IFindPairsProcessor` first checks if they are actually colliding using
`Physics.DistanceBetween()`. If that check passes, it is allowed to modify the
health of both entities.

```csharp
struct DamageCollidingShipsProcessor : IFindPairsProcessor
{
    public PhysicsComponentLookup<ShipHealth> shipHealthLookup;
    [ReadOnly] public ComponentLookup<Damage> shipDamageLookup;

    public void Execute(in FindPairsResult result)
    {
        if (Physics.DistanceBetween(result.colliderA, result.transformA, result.colliderA, result.transformA, 0f, out _))
        {
            ref var healthA = ref shipHealthLookup.GetRW(result.entityA).ValueRW;
            ref var healthB = ref shipHealthLookup.GetRW(result.entityB).ValueRW;
                    
            var damageA = shipDamageLookup[result.entityA];
            var damageB = shipDamageLookup[result.entityB];

            healthA.health -= damageB.damage;
            healthB.health -= damageA.damage;
        }
    }
}
```

Unlike with `ComponentLookup`, `PhysicsComponentLookup` requires the entity
passed in be a `SafeEntity`. The `FindPairsResult` properties `entityA` and
`entityB` are of this type. `SafeEntity` can be implicitly casted to a normal
Entity, but you can’t go the other way. Additionally, if safety checks are
enabled, an exception will be thrown when a `SafeEntity` is used in a
`PhysicsComponentLookup` inside a `RunImmediate()` or `ScheduleParallelUnsafe()`
context.

You can assign a regular `ComponentLookup` to a `PhysicsComponentLookup`, making
setup of the processor easy.

```csharp
var processor = new DamageCollidingShipsProcessor
{
    shipHealthLookup = SystemAPI.GetComponentLookup<ShipHealth>(),
    shipDamageLookup = SystemAPI.GetComponentLookup<Damage>()
};
```

Similar to `PhysicsComponentLookup`, there is also a `PhysicsBufferLookup` type
for dynamic buffers.

## FindPairs with Two Collision Layers

Being able to find all pairs within a `CollisionLayer` is useful. However, it
isn’t that common of a use case. Typically, each `CollisionLayer` represents a
specific kind of thing. And the interesting stuff happens when **different**
kinds of things interact. Instead of finding pairs within a `CollisionLayer`,
you may want to find pairs between two `CollisionLayers`. FindPairs can do this
too.

To find pairs between two `CollisionLayer` instances, simply pass in both
instances into `Physics.FindPairs()` along with the `IFindPairsProcessor`
instance. **Order is important.** The first layer passed in becomes `layerA`,
and the second layer becomes `layerB`. For every pair, everything with the `A`
suffix comes from `layerA`, and everything with the `B` suffix comes from
`layerB`. You can make aggressive assumptions this way.

FindPairs with two layers will use two threads (three when safety is enabled)
for the second phase.

## Modifying ColliderBody Data

FindPairs always treats the `CollisionLayers` you pass in as `[ReadOnly]`. You
are not allowed to modify data in the `CollisionLayer` at runtime. However, that
doesn’t mean you can’t work around this.

The data stored in the `CollisionLayer` is a copy of the source data, and does
not automatically synchronize. Therefore, you can choose to use the actual
source data instead. For example, you could use a
`PhysicsComponentLookup<Collider>` instead of the stored collider. For
transforms, you can obtain mutable access to the `TransformAspect` via
`PhysicsTransformAspectLookup`.

**Warning: FindPairs will NOT update AABBs in the middle of a FindPairs
operation. If you intend to modify your colliders and transforms mid-search, you
should pre-expand your AABBs when building the** `CollisionLayer`**.**

## Other Utilities

While write-safety is one of the strong points of FindPairs, sometimes you may
still wish to use it for its performance benefits or clean API structure even
when you don’t need write-access to both entities involved in a pair. FindPairs
has more to offer even in these scenarios.

You can disable safe write access to `layerB` entirely in a two-layer FindPairs
by using `ScheduleParallelByA()`. This variant can do all of its work in a
single phase. Or for complete parallelism, you can disable write safety
completely via `ScheduleParallelUnsafe()`. This one is useful if all you need to
do are force `bool` values or `IEnableableComponent` in a specific direction.
You can also use this for command buffers using `FindPairsResult.jobIndex`. And
you can write to a `PairStream.ParallelWriter` in this mode using
`FindPairsResult.pairStreamKey`.

For even more advanced use cases, `IFindPairsProcessor` has two default methods
you can override. They are `BeginBucket()` and `EndBucket()`. These give you a
`FindPairsBucketContext` which provide access to all the layer elements that
would be processed for the specific `jobIndex`. Combined with
`Physics.FindPairsJobIndexCount()`, you can use these to populate a
`NativeStream` or configure other caches.

## What’s Next

Psyshock has a novel approach to physics, being inside-out in its design and
leaving you in control over your data. There are plenty of additional APIs to
explore, such as a player input integrator for smoother player motion, ballistic
utilities for particle physics, and ForEachPair using `PairStreams` to cache
data with entity pairs and process them multiple times later. For full rigid
body simulation, you can check out [UnitySim](UnitySim.md).

Psyshock can be intimidating. No question is too stupid and you are encouraged
to reach out to the community on Discord if you want help making things work or
would like to get more performance out of your use case.
