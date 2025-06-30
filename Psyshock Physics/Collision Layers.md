# Collision Layers and Collision Worlds

A `CollisionLayer` is a struct containing several native containers and
properties. It is used by the broadphase algorithms to provide fast and
thread-safe collision detection and event handling.

A `CollisionWorld` is a specialized `CollisionLayer` with additional cached
entity archetype info so that entity queries can be applied on top of spatial
queries efficiently

## The Collision Layer Structure

A collision layer does not use any pointers, nor does the broadphase algorithm
use any `ComponentLookup<>` to perform its work. Instead, all data is copied
upon construction. The following pieces of data are stored in a
`CollisionLayer`:

-   Aabb
-   ColliderBody – a struct which contains the following
    -   Collider
    -   RigidTransform – collider-space to world-space
    -   Entity – or some other 64bit ID encoded into an `Entity` struct
-   Source Index – an integer used to remap elements to an originating array

Collision layers are immutable, which means that if any of the above properties
change, a new `CollisionLayer` must be built.

A collision layer reorders the above data to a cache-optimized form. It assigns
each element an index based on this new ordering scheme. The sourceIndices
property provides the array of the original indices sorted by the order they
exist in the `CollisionLayer`.

### Collision Layer Settings

A `CollisionLayerSettings` object can be used to customize the structure of a
`CollisionLayer`. The settings **must be identical** when multiple collision
layers are used together.

A `CollisionLayerSettings` contains the following:

-   worldAabb – The axis-aligned box which defines the subdivision offsets of
    the multi-box broadphase cells
-   worldSubdivisionsPerAxis – The number of subdivisions of the `worldAabb` by
    axis

Note: The `worldAabb` does **not** need to fully encapsulate all colliders. The
outer surface cells within the `worldAabb` are “open” and extend off to
infinity. The “extents” of the `worldAabb` have no effect for subdivisions of 2
or less per corresponding axis.

If no `CollisionLayerSettings` is provided to the builder, the layer will be
built centered about the origin and with `worldSubdivisionsPerAxis` set to (2,
2, 2).

## Building a Collision Layer

Collision layers are built using a fluent builder API. The API provides several
customizations to meet different performance needs.

### Step 1: Physics.BuildCollisionLayer – Choose your data

You start the fluent chain by calling `Physics.BuildCollisionLayer`. This method
has three overloads. All overloads return a `BuildCollisionLayerConfig`. The
config does not internally allocate any native containers, so it is safe to
discard.

#### (EntityQuery query, BuildCollisionLayerTypeHandles handles)

This overload allows you to build a `CollisionLayer` from an `EntityQuery`.
Currently, the only requirement of the `EntityQuery` is that it specifies a
`Collider` in the “All” category. If using a `FluentQuery`, you can ensure
correctness by calling ` PatchQueryForBuildingCollisionLayer()` in the
`FluentQuery` chain.

The `handles` argument must be provided by an actively running system. The
builder uses this to obtain necessary `ComponentTypeHandle`s. It obtains the
following read-only handles:

-   Collider
-   Entity
-   WorldTransform / LocalToWorld

You should cache these handles inside of `OnCreate()` and remember to `Update()`
them each `OnUpdate()`.

#### (NativeArray/List\<ColliderBody\> bodies)

This overload allows you to build a `CollisionLayer` from the set of raw data at
runtime. You can use this to procedurally generate collider data that will be
stored in the `CollisionLayer`.

Each `Aabb` is calculated from the `Collider` and `RigidTransform` of each
element.

The `NativeList` variant will invoke `AsDeferredJobArray()` allowing for an
unknown number of elements at schedule-time.

#### (NativeArray/List\<ColliderBody\> bodies, NativeArray/List\<Aabb\> overrideAabbs)

This overload is similar to the previous overload, except instead of calculating
the `Aabb`s from the `Collider` and `RigidTransform`, it uses the passed in
`Aabb`s instead. This allows you to do more custom operations such as accounting
for projected movements. Use the `Physics.AabbFrom()` and
`UnitySim.MotionExpansion.ExpandAabb()` methods for common use cases.

### Step 2: WithXXX - Choose your settings

This step is completely optional. However, some of these settings may provide
you major performance improvements or provide you access to extra metadata about
the build process.

You may invoke multiple settings in your fluent chain.

#### WithSettings(CollisionLayerSettings settings)

This method sets the `CollisionLayerSettings` from a preconstructed object. This
is most useful when storing a `CollisionLayerSettings` in a component on the
`sceneGlobalEntity`.

#### WithWorldBounds(Aabb worldAabb)

This method sets the `worldAabb` property of the `CollisionLayerSettings`.

#### WithSubdivisions(int3 subdivisions)

This method sets the `worldSubdivisionsPerAxis` property of the
`CollisionLayerSettings`.

#### WithWorldMin(float x, float y, float z)

This method sets the `min` values of `worldAabb` property of the
`CollisionLayerSettings`. This method should primarily be used for prototyping
purposes.

#### WithWorldMax(float x, float y, float z)

This method sets the `max` values of `worldAabb` property of the
`CollisionLayerSettings`. This method should primarily be used for prototyping
purposes.

### Step 3: RunXXX / ScheduleXXX – Choose your scheduler

The final step is to choose how to schedule the construction of the collision
layer. Construction can be expensive, especially with large input data.

All of these methods specify an `Allocator` which will be used for all native
containers inside the `CollisionLayer`. Custom allocators are supported.

#### RunImmediate(out CollisionLayer layer, Allocator allocator)

This mode does not use any jobs whatsoever. Its purpose is to be executed in the
context of an `IJob` with Burst.

This method does not work with `EntityQuery` data.

#### Run(out CollisionLayer layer, Allocator allocator)

This mode constructs the `CollisionLayer` on the main thread using Burst.

#### ScheduleSingle(out CollisionLayer layer, Allocator allocator, JobHandle inputDeps = default)

This mode constructs the `CollisionLayer` using single-threaded jobs. It returns
a JobHandle that must be completed or used as a dependency for any operation
that wishes to use the output `layer` or `remapSrcArray`.

For `EntityQuery` data, multiple jobs are scheduled and may appear on different
threads due to Unity’s job scheduler.

For array data, only one job is scheduled.

#### ScheduleParallel(out CollisionLayer layer, Allocator allocator, JobHandle inputDeps = default)

This mode constructs the `CollisionLayer` using parallel jobs for as many stages
of the construction as possible. It returns a `JobHandle` that must be completed
or used as a dependency for any operation that wishes to use the output `layer`
or `remapSrcArray`.

You may see these stages in the profiler:

-   Part 1
    -   From Query – `IJobChunk` parallelism
    -   From Collider Body Arrays – `IJobFor` with batch count of 64
    -   From Dual Arrays - `IJobFor` with batch count of 64
-   Part 2 - `IJob`
-   Part 3 - `IJobFor` with batch count of 512
-   Part 4 - `IJobFor` with batch count of 1 (per bucket)
-   Part 5 - `IJobFor` with batch count of 128

With the exception of the job types themselves, there is no algorithmic overhead
for using parallel jobs. All algorithms used in `CollisionLayer` construction
are O(n).

There is also no memory overhead for using parallel jobs, although this is more
of a lack of reusing temp memory in the single-threaded use-case.

### Sort your Data

In case you want to reorder your data such that it aligns with the
`CollisionLayer`, here’s an example of how to do it:

```csharp
Physics.BuildCollisionLayer(query, this).Run(out var layer, state.WorldUpdateAllocator);
var transforms = query.ToComponentDataArray<WorldTransform>(state.WorldUpdateAllocator);
var reorderedTransforms = new NativeArray<WorldTransform>(transforms.Length, state.WorldUpdateAllocator, NativeArrayOptions.UninitializedMemory);
for (int i = 0; i < reorderedTransforms.Length; i++)
{
    reorderedTransforms[i] = transforms[layer.sourceIndices[i]];
}
```

## FindPairs – Executing a Broadphase Query

`FindPairs` is the name of the broadphase algorithm in Psyshock. It allows you
to immediately react to the pairs it finds in a thread-safe manner directly
within the algorithm. It uses generic jobs and a clever Fluent API dispatcher to
operate.

`FindPairs` can operate in two different modes: self and bipartite

#### Self

In self mode, `FindPairs` finds all pairs of overlapping `Aabb`s within a single
`CollisionLayer`. Whether an element in a pair is “first” (denoted as “A”) or
“second” (denoted as “B”) is deterministic but arbitrary.

#### Bipartite

In bipartitie mode, `FindPairs` finds all pairs of overlapping `Aabb`s between
two `CollisionLayer`s. In other words, it only reports pairs with one element in
the first layer, and one element in the second layer. The first element in any
pair (the “A” element) is always from the first layer (“layerA”). Likewise, the
second element in any pair (the “B” element) is always from the second layer
(“layerB”). This rule allows you to make aggressive assumptions about each pair
of elements found.

### IFindPairsProcessor

`IFindPairsProcessor` is an interface you must implement in a struct and provide
an instance of when invoking `FindPairs`. Think of it like a custom job type,
except you do not need to add the `[BurstCompile]` attribute.

The interface requires that you implement an `Execute(in FindPairsResult
result)`.

#### FindPairsResult

`FindPairsResult` is the data the `FindPairs` algorithm provides you whenever it
finds a pair of overlapping `Aabb`s. It contains the following properties:

-   entityA – A key into `PhysicsComponentLookup` which corresponds to
    `bodyA.entity`
-   entityB - A key into `PhysicsComponentLookup` which corresponds to
    `bodyB.entity`
-   colliderA – The first collider element in the pair
-   colliderB – The second collider element in the pair
-   transformA – The first transform element in the pair
-   transformB – The second transform element in the pair
-   bodyA – The `ColliderBody` representing the first element in the pair
-   bodyB – The `ColliderBody` representing the second element in the pair
-   aabbA – The `Aabb` of the first element in the pair
-   aabbB – The `Aabb` of the second element in the pair
-   layerA – The full `CollisionLayer` the first element in the pair comes from
-   layerB – The full `CollisionLayer` the second element in the pair comes from
-   bodyIndexA – The index of the first element in the pair as stored in its
    `CollisionLayer`
-   bodyIndexB – The index of the second element in the pair as stored in its
    `CollisionLayer`
-   sourceIndexA – The index of the first element in the pair relative to the
    `CollisionLayer`’s created source.
-   sourceIndexB – The index of the second element in the pair relative to the
    `CollisionLayer`’s created source.
-   jobIndex – An index for deterministically writing to
    `EntityCommandBuffer.ParallelWriter` and similar API
-   pairStreamKey – A special key which can be used for writing to a
    `PairStream.ParallelWriter`

#### PhysicsComponentLookup and SafeEntity

In `FindPairs`, the two entities in the result passed into the
`IFindPairsProcessor` are guaranteed to be thread-safe writable. In other words,
you can safely write to any of those entities components without race
conditions. However, Unity’s default safety system for `ComponentLookup` doesn’t
understand this.

If you need to do this writing, you can instead use `PhysicsComponentLookup`.
You assign this a normal `ComponentLookup` when constructing your
`IFindPairsProcessor`. `PhysicsComponentLookup` works just like
`ComponentLookup` except instead of being indexed with `Entity`, it is indexed
using `SafeEntity`. The only way to obtain a `SafeEntity` is from the `entityA`
and `entityB` properties of a `FindPairsResult`.

**Warning: This safety rule using SafeEntity only applies if only one occurrence
of an Entity exists in the CollisonLayer(s) passed to FindPairs. Do not use
parallel FindPairs if this rule cannot be met.**

When safety checks are enabled, an error will be thrown if the above rule is
violated.

#### Indices

Similar to how the two entities’ components are thread-safe writable when
processing a `FindPairsResult`, any `NativeArray` element at either `bodyAIndex`
or `bodyBIndex` is also thread-safe writable. However, unlike
`PhysicsComponentLookup`, there is no thread-safe wrapper for `NativeArray`s.

`jobIndex` is **not** unique per `FindPairsResult`. It is unique to each *cell
vs cell query* the `FindPairs` algorithm performs.

#### Narrow-Phase

While you can perform any logic and checks inside the `Execute` method of an
`IFindPairsProcessor`, a common check is to determine if the two colliders are
really intersecting. You can do that with this expression:

```csharp
if (Physics.DistanceBetween(result.bodyA.collider, result.bodyA.transform, result.bodyB.collider, result.bodyB.transform, 0f, out _))
```

#### Buckets

For advanced use cases, you can additionally implement the `BeginBucket()` and
`EndBucket()` interface methods in the `IFindPairsProcessor`. These combined
with `Physics.FindPairsJobIndexCount()` can be used to properly set up and use
`NativeStream.Writer` in an `IFindPairsProcessor`.

## Running FindPairs

Similar to building `CollisionLayer`s, `FindPairs` is also invoked using a
fluent API.

### Step 1: Physics.FindPairs – Choose your data

You start a fluent chain using `Physics.FindPairs()`. This returns a
`FindPairsConfig<T>` where `T` is your `IFindPairsProcessor` type you pass in as
the final argument.

There are two overloads:

#### (CollisionLayer layer, T processor)

This overload specifies that you want to run `FindPairs` in “self” mode using
the single `layer`. It will find all overlapping `Aabb`s in `layer` and report
them to `processor`.

#### (CollisionLayer layerA, CollisionLayer layerB, T processor)

This overload specifies that you want to run `FindPairs` in “bipartite” mode
using `layerA` and `layerB`. It will find all instances where an `Aabb` in
`layerA` overlaps an `Aabb` in `layerB` and report such instances to
`processor`. For each `result` passed to `processor`, the first element (“A”
element) always comes from `layerA` and the second element (“B” element) always
comes from `layerB`.

### Step 2: WithXXX – Choose your options

You can customize the algorithm to improve performance or disable specific
safety checks like entity aliasing.

#### WithoutEntityAliasingChecks()

Component Safety guarantees on found pairs are invalid if an entity exists
multiple times in a `CollisionLayer`. If you want to disable the safety check
for this in parallel jobs, use this method.

#### WithCrossCache()

FindPairs is typically a two-phased algorithm. The second phase is limited in
the number of threads it can utilize. This method moves some of the computation
of the second phase to the highly parallel first phase, at the cost of needing
to store the results of the computation in an intermediate buffer. This
sacrifices some throughput to reduce latency under heavy loads.

### Step 3: RunXXX / ScheduleXXX – Choose your scheduler

While `FindPairs` is an optimized algorithm, it can still be computationally
expensive, especially with complex `IFindPairsProcessor`s.

#### RunImmediate()

This mode does not use any jobs whatsoever. Its purpose is to be executed in the
context of an `IJob` with Burst.

#### Run()

This mode executes the `FindPairs` algorithm on the main thread using Burst.

#### ScheduleSingle(JobHandle inputDeps = default)

This method schedules the `FindPairs` algorithm on a single thread. Only one job
thread is scheduled. The method returns a `JobHandle` for the scheduled job.

#### ScheduleParallel(JobHandle inputDeps = default)

This method schedules the `FindPairs` algorithm using multiple jobs for as many
stages as possible. The method returns a `JobHandle` which represents the final
state of the `FindPairs` algorithm.

You may see these stages in the profiler:

-   Part 1 – Parallel job with a batch count of 1 (per bucket)
-   Part 2 Self – Single-threaded job
-   Part 2 Bipartite – Dual-threaded job
-   Part 2 With Safety – When safety checks are enabled, this variant invokes an
    additional worker thread to validate that no entity aliasing has occurred
    between the layers

*Note: You can disable the entity aliasing check using
WithoutEntityAliasingChecks() between steps 1 and 2.*

#### ScheduleParallelByA(JobHandle inputDeps = default)

This method schedules the `FindPairs` algorithm using a single job, and only
provides thread-safety for the first element in each pair in a bipartite
operation.

#### ScheduleParallelUnsafe(JobHandle inputDeps = default)

This method schedules the `FindPairs` algorithm using multiple jobs for all
stages but sacrifices thread safety for the pair of entities and indices.

There are a few applications where this is useful. First, if the processor may
only ever write a hardcoded value and never read from it, then the operation is
deterministic regardless of the order of the writes. Second, `jobIndex` is still
safe, which means writing to `EntityCommandBuffer.ParallelWriter` is also safe.
Third, `pairStreamKey` is also safe, allowing for fast storage into
`PairStreams`

This method returns a `JobHandle `which represents the final state of the
`FindPairs` algorithm.

## FindObjects

If you only need to query a single `Aabb` against a `CollisionLayer`, you can
use the `FindObjects` API. It works very similarly to `FindPairs`, but with
fewer options. Though it also has a zero-option overload for receiving each
result in a `foreach` expression, which can be very convenient.

## CollisionWorld

The `CollisionWorld` type is very similar to `CollisionLayer` in terms of APIs.
It is composed of a `CollisionLayer`, a list of `EntityArchetype` captured from
the entities at build time, and a mapping between the two.

Unlike with `CollisionLayer`, a `CollisionWorld` may only be created from an
`EntityQuery`. You cannot use arrays or lists. To override AABBs, you must
populate the optional `WorldCollisionAabb` component for each entity you wish to
override.

Every `CollisionWorld` can be built with a specified world index via the
`WithWorldIndex()` in the fluent builder API. This value is exclusively
user-defined. It is written to the optional `WorldCollisionIndex` along with the
body index when building a `CollisionWorld`. `WorldCollisionIndex` provides a
convenient way to look up the body index in a `CollisionWorld` from the entity.

In addition to the `CollisionLayer` properties, `CollisionWorld` provides a
couple additional properties. `archetypeCount` contains the number of unique
archetypes that were found on entities when building the `CollisionWorld`. And
`archetypeIndices` is a per-body array of indices to a unique archetype. Bodies
which share the same archetype index have the same archetype.

### Masks

The `CollisionWorld.Mask` is an `EntityQuery` evaluated on the `CollisionWorld`.
The `Mask` contains all archetype indices in the `CollisionWorld` that match the
entity query. This is backed by `Allocator.Temp` memory for a sufficiently large
set of matching archetypes, and therefore `Mask` should not be passed between
jobs or persist for multiple frames.

You create a `Mask` using `CollisionWorld.CreateMask()`. You can specify either
a `TempQuery` or an `EntityQueryMask`. The `Mask` can be used in a `foreach` to
iterate the archetype indices directly if you ever need that. However, the
common use case is to pass it into spatial query methods such as `Raycast()` or
`DistanceBetween()`.

### FindPairs

You can perform FindPairs operations with `CollisionWorld` instances and Masks.
There are special rules for which pairs get reported:

-   Single `CollisionWorld`, single `Mask` – Each unique pair reported once.
-   Single `CollisionWorld`, `Mask`s A and B – Each pair is composed of an
    entity matching `Mask` A as `entityA`, and an entity matching `Mask` B as
    `entityB`. If a pair of entities match both masks, then the pair may be
    reported twice, with each entity in the pair being reported as `entityA`
    once and `entityB` the other time. However, an entity will never be in a
    pair with itself.
-   Two `CollisionWorlds`, `Mask`s A and B – Each `CollisionWorld` is filtered
    by its own respective mask, and otherwise the FindPairs operates using the
    standard bipartite rules. An entity may not be present in both
    `CollisionWorld` instances.

## Debugging Collision Layers

`Psyshock` provides a Fluent API for visualizing collision layers and
hypothetical FindPairs operations. These APIs use `UnityEngine.Debug.DrawLine()`
internally. There is a limit to how many lines Unity can draw. The limit can be
customized in the command line arguments which you can configure from Unity Hub.

### Drawing a layer

`PhysicsDebug.DrawLayer()` is the entry-point for visualizing a
`CollisionLayer`. It draws out the AABBs for each collider using a different
color for each grid cell. You can customize the colors using `WithColors()`. The
second argument `crossBucketColor` represents the color for colliders which span
multiple grid cells.

### Drawing a hypothetical FindPairs

`PhysicsDebug.DrawFindPairs()` is the entry-point for visualizing a
`CollisionLayer`. It draws out the AABBs for each collider using one of two
colors depending on if the collider’s AABB overlaps another. Self and Bipartite
rules apply. `WithColors()` can set the overlap and non-overlap colors as well
as disable drawing non-overlapping colliders.

### Draw an AABB

If you need a Burst-safe method to draw an AABB just like the previous methods,
you can use `PhysicsDebug.DrawAABB()`.

### Draw a Collider

If you need a Burst-safe method to draw a transformed collider, you can use
`PhysicsDebug.DrawCollider()`. Some overloads contain a `segmentsPerPi` argument
for specifying the amount of detail to be drawn for round collider types.
