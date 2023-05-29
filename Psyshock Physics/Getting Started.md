# Getting Started with Psyshock Physics

This is the seventh preview version I am releasing out to the public. It
currently only supports a small number of use cases. The number of supported use
cases will grow with each release.

## Authoring

Currently Psyshock uses the classical Physx components for authoring
[colliders](Colliders.md). Attach a Sphere Collider, Capsule Collider, Box
Collider, or Mesh Collider to the GameObject you wish to convert. Mesh Colliders
must have their convex checkbox checked.

There’s one exception. I added support for Compound Colliders and this uses the
new Collider Authoring component. But you can also create them by attaching more
than one Physx collider component to a Game Object.

## Colliders in code

```csharp
//How to create Collider

//Step 1: Create a sphere collider
var sphere = new SphereCollider(float3.zero, 1f);

//Step 2: Assign it to a collider
Collider collider = sphere;

//Step 3: Attach the collider to an entity
EntityManager.AddComponentData(sceneGlobalEntity, collider);

//How to extract sphere collider

//Step 1: Check type
if (collider.type == ColliderType.Sphere)
{
    //Step 2: Assign to a variable of the specialized type
    SphereCollider sphere2 = collider;

    //Note: With safety checks enabled, you will get an exception if you cast to the wrong type.
}

//EZ PZ, right?
```

## Simple Queries

-   Current
    -   Physics. AabbOf
    -   Physics.Raycast
    -   Physics.DistanceBetween
    -   Physics.ColliderCast
-   Future
    -   Physics.AreIntersecting
    -   Physics.ComputeContacts
    -   Physics.QuadraticCast
    -   Physics.QuadraticColliderCast

*Warning: Unlike Unity Physics,* `Raycast()` *and* `ColliderCast()` *do not
report inside hits or “zero-distance” hits. Use a* `DistanceBetween()` *query at
the start point to get inside hit info.*

## Simple Modifiers

-   Current
    -   Physics.ScaleStretchCollider
    -   Physics.CombineAabb

## Scheduling Jobs

Some Physics operations allow for scheduling jobs (as well as executing directly
in a job). These operations expose themselves through a Fluent API with a
specific layout:

```csharp
Physics.SomePhysicsOperation(requiredParams).One(or).MoreExtra(Settings).Scheduler();
```

## Building Collision Layers

[Collision layers](Collision%20Layers.md) represent groups of colliders that can
be queried in the world. They can be generated from entities or some other
simulation data. Collision layer construction uses a Fluent API.

You start the Fluent chain by calling `Physics.BuildCollisionLayer()`. There are
a couple of variants based on whether you want to build from an `EntityQuery` or
`NativeArray`s.

If you build from an `EntityQuery`, the `EntityQuery` must have the `Collider`
component. If you use `SubSystem.Fluent`, you can use the extension method
`PatchQueryForBuildingCollisionLayer()` when building your `EntityQuery`.

Next in the fluent chain, you have the ability to apply custom settings and
options. You can customize the `CollisionLayerSettings` for better performance.
The settings are given as follows:

-   worldAabb – The `Aabb` from which to construct the multibox
-   worldSubdivisionsPerAxis – How many “buckets” AKA cells to divide the world
    into along each axis

*Important:* `worldAabb` *and* `worldSubdivisionsPerAxis` *MUST match when
performing queries across multiple* `CollisionLayer`*s. A safety check exists to
detect this when safety checks are enabled.*

Lastly, after customizing things the way you want, you can call one of the many
schedulers:

-   .RunImmediate – Run without scheduling a job. You can call this from inside
    a job. This does not work using an `EntityQuery` as the source.
-   .Run – Run on the main thread with Burst.
-   .ScheduleSingle – Run on a worker thread with Burst.
-   .ScheduleParallel – Run on multiple worker threads with Burst.

*Feedback Request: Anyone have a better name for
PatchQueryForBuildingCollisionLayer?*

## Using Collision Layers

Collision Layers can be passed into jobs as `[ReadOnly]` and queried. You can
fire raycasts at them, or colliders, or perform distance queries. You can also
iterate through the colliders and all their data via the `colliderBodies`
property. Note that these bodies will be spatially reordered relative to how
they were created. Use the `srcIndices` property to get the original creation
index (`entityInQueryIndex` or `ColliderBody` array index).

*Tip: Store a* `CollisionLayer` *associated with each* `EntityQuery` *you care
about inside an* `ICollectionComponent` *on the* `sceneBlackboardEntity`*. This
will allow multiple systems to use the same* `CollisionLayer` *for the same
query without having to rebuild the* `CollisionLayer`*.*

## Using FindPairs

`FindPairs` is a broadphase algorithm that lets you immediately process pairs
with any narrow phase or other logic you desire. `FindPairs` has unique
threading properties that ensures the two Entities in the pair can be modified
with `PhysicsComponentLookup` which can be implicitly constructed from a
`ComponentLookup`.

`FindPairs` uses a fluent syntax. Currently, there are two steps required.

-   Step 1: Call `Physics.FindPairs`
    -   To find pairs within a layer, only pass in a single `CollisionLayer`.
    -   To find pairs between two layers (Bipartite), pass in both layers. No
        pairs are generated within each individual layer. It is up to you to
        ensure no entity exists simultaneously in both layers or else scheduling
        a parallel job will not be safe. When safety checks are enabled, an
        exception will be thrown if this rule is violated.
    -   The final argument in `FindPairs` is an `IFindPairsProcessor`. Implement
        this interface as a struct containing any native containers your
        algorithm relies upon. When done correctly, your struct should resemble
        a non-lambda job struct.
-   Step 2: Call a scheduling method
    -   .RunImmediate – Run without scheduling a job. You can call this from
        inside a job.
    -   .Run – Run on the main thread with Burst.
    -   .ScheduleSingle – Run on a worker thread with Burst.
    -   .ScheduleParallel – Run on multiple worker threads with Burst.
    -   .ScheduleParallelUnsafe – Run on multiple worker threads with maximum
        parallelization. This is unsafe and should only be used for marking
        colliders as hit or for writing to an
        `EntityCommandBuffer.ParallelWriter`
