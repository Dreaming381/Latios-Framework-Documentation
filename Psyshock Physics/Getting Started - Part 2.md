# Getting Started with Psyshock Physics – Part 2

In the previous part, we explored the collider types and the operations that
could be performed on them. In this part, we’ll explore *collision layers*.

## What is a Collision Layer?

[Collision layers](Collision%20Layers.md) represent groups of colliders that can
be queried in the world. They are native container collections, just like
`NativeList` and `NativeHashMap` (actually, they are a struct with multiple
containers inside). You can think of a `CollisionLayer` as a data structure used
for fast queries, like an octree or a BVH.

Collision layers do NOT have any relationship with Game Object layers, and are
never created automatically.

A collision layer is composed of flat arrays in which the elements are grouped
in intervals called “buckets”. The collision layer has a course fixed grid
within an axis-aligned bounding box. In PhysX, this is referred to as a
“multi-box”. Each “cell” within this grid corresponds to a bucket.

The configuration for the bounding box and its subdivisions are specified by the
`CollisionLayerSettings`:

```csharp
public struct CollisionLayerSettings
{
    /// <summary>
    /// An AABB which defines the bounds of the subdivision grid.
    /// Elements do not necessarily need to fit inside of it.
    /// </summary>
    public Aabb worldAabb;
    /// <summary>
    /// How many "cells" to divide the worldAabb into.
    /// </summary>
    public int3 worldSubdivisionsPerAxis;
}
```

Cells on the boundary of the layer’s bounding box are “open” and extend off to
infinity. If a collider touches multiple cell volumes, it is instead placed in
the “cross-bucket”, which is a special bucket. Colliders with NaN data will be
placed in a NaN bucket which is omitted from queries.

Inside each bucket, elements are sorted relative to the x-axis.

### Tuning for Performance

The purpose of the multi-box structure is entirely for performance reasons. Any
configuration will produce logically correct results, but there is typically a
configuration that is optimal.

The most common issue people run into is that one of the division planes between
cells cuts across nearly all colliders. An example of this would be a division
along y=0 in a top-down game where the ground surface is placed near y=0.
Improper division placement may result in most colliders being grouped into the
cross-bucket, which degrades performance. Spacing cells smaller than the typical
collider size can also cause this problem.

Due to the boundary cell rules, you want your layer’s bounding box to
encapsulate the **majority** of the **action**.

Lastly, you should only ever raise the subdivisions of the x-axis when you are
having trouble increasing y and z and need more parallelism. This is because the
x-axis already benefits from sorting, and the subdivisions don’t offer much
additional benefits.

Here’s some examples:

-   A top-down game with a large mostly-flat world
    -   (1, 1, 8) or (2, 1, 5) or similar
-   A 2D side-scrolling platformer
    -   (3, 4, 1) or similar
-   An arena shooter with lots of verticality and lots of bullets
    -   (2, 3, 3) or (1, 4, 4) or similar
-   A massive space battle
    -   (1, 12, 12) or similar

The default collision layer settings use (2, 2, 2) centered at the origin.

## A Note About Job-Scheduling APIs

Some Physics operations allow for scheduling jobs (as well as executing directly
in a job). These operations expose themselves through a Fluent API with a
specific layout:

```csharp
Physics.SomePhysicsOperation(requiredParams).One(or).MoreExtra(Settings).Scheduler();
```

## Building Collision Layers

You can create collision layers from entity queries, or from your own list of
colliders. Collision layer construction uses a Fluent API.

You start the Fluent chain by calling `Physics.BuildCollisionLayer()`. There are
a couple of variants based on whether you want to build from an `EntityQuery` or
`NativeArray`s.

If you build from an `EntityQuery`, the `EntityQuery` must have the `Collider`
component and the appropriate world-space transform for the transform system you
are using (`WorldTransform` for QVVS or `LocalToWorld` for Unity Transforms). If
you use `FluentQuery`, you can use the extension method
`PatchQueryForBuildingCollisionLayer()` when building your `EntityQuery`.
Additionally, when inside an `ISystem`, you will need to provide a
`BuildCollisionLayerTypeHandles`. These must be cached in `OnCreate()` and
updated before each use.

If you build from arrays, you need to provide a `NativeArray<ColliderBody>`.
`ColliderBody` contains an `Entity`, a `Collider`, and a `TransformQvvs`. You
can optionally also pass a `NativeArray<Aabb>` to override the bounds of the
colliders. This is useful if you want to expand the bounds to account for
motion.

Next in the fluent chain, you can optionally provide custom settings and
options. With `WithSettings()`, can customize the `CollisionLayerSettings` for
better performance.

*Important:* `worldAabb` *and* `worldSubdivisionsPerAxis` *MUST match when
performing queries across multiple* `CollisionLayer`*s (more on that in Part 3).
A safety check exists to detect this when safety checks are enabled.*

Lastly, after customizing things the way you want, you can call one of the many
schedulers:

-   .RunImmediate – Run without scheduling a job. You can call this from inside
    a job. You cannot use an `EntityQuery` as the source for this scheduler.
-   .Run – Run on the main thread with Burst.
-   .ScheduleSingle – Run on a worker thread with Burst.
-   .ScheduleParallel – Run on multiple worker threads with Burst.

*Feedback Request: Anyone have a better name for
PatchQueryForBuildingCollisionLayer?*

### Sharing a Collision Layer Between Systems

If you only need the Collision Layer within the same system you build it, it is
usually best to allocate it using `WorldUpdateAllocator`. However, if you wish
for other systems to use it, a great way option is to store it on the
`sceneBlackboardEntity` as a collection component. Use the `OnNewScene()`
callback to add the collection component the first time and avoid a sync point.

### BuildCollisionLayer Examples

Let’s start with building a collision layer using an `EntityQuery` for all
entities tagged as environment and storing it on the `sceneBlackboardEntity`.

```csharp
public struct EnvironmentTag : IComponentData { }

public partial struct EnvironmentCollisionLayer : ICollectionComponent
{
    public CollisionLayer layer;

    public JobHandle TryDispose(JobHandle inputDeps) => layer.IsCreated ? layer.Dispose(inputDeps) : inputDeps;
}
[BurstCompile]
public partial struct BuildEnvironmentCollisionLayerSystem : ISystem, ISystemNewScene
{
    LatiosWorldUnmanaged           latiosWorld;
    BuildCollisionLayerTypeHandles m_handles;
    EntityQuery                    m_environmentQuery;

    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        latiosWorld = state.GetLatiosWorldUnmanaged();
        m_handles   = new BuildCollisionLayerTypeHandles(ref state);

        m_environmentQuery = state.Fluent().With<EnvironmentTag>(true).PatchQueryForBuildingCollisionLayer().Build();
    }

    public void OnNewScene(ref SystemState state)
    {
        latiosWorld.sceneBlackboardEntity.AddOrSetCollectionComponentAndDisposeOld<EnvironmentCollisionLayer>(default);
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        m_handles.Update(ref state);

        var settings = new CollisionLayerSettings { worldAabb = new Aabb(-300f, 300f), worldSubdivisionsPerAxis = new int3(3, 4, 4) };
            
        state.Dependency = Physics.BuildCollisionLayer(m_environmentQuery, in m_handles).WithSettings(settings)
                            .ScheduleParallel(out var environmentLayer, Allocator.Persistent, state.Dependency);
            
        latiosWorld.sceneBlackboardEntity.SetCollectionComponentAndDisposeOld(new EnvironmentCollisionLayer { layer = environmentLayer });
    }
}
```

Here’s an example which builds the layer using procedural data. In this case, it
is building a capsule collider whose inner segment endpoints connect the
entity’s previous position and current position:

```csharp
[BurstCompile]
public partial struct BuildBulletsCollisionLayerSystem : ISystem, ISystemNewScene
{
    LatiosWorldUnmanaged latiosWorld;

    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        latiosWorld = state.GetLatiosWorldUnmanaged();
    }

    public void OnNewScene(ref SystemState state) => latiosWorld.sceneBlackboardEntity.AddOrSetCollectionComponentAndDisposeOld(new BulletCollisionLayer());

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        CollisionLayerSettings settings = latiosWorld.sceneBlackboardEntity.GetComponentData<ArenaCollisionSettings>().settings;

        var query = QueryBuilder().WithAll<WorldTransform, Collider, PreviousTransform, BulletTag>().Build();

        var bodies =
            CollectionHelper.CreateNativeArray<ColliderBody>(query.CalculateEntityCount(),
                                                                state.WorldUnmanaged.UpdateAllocator.ToAllocator,
                                                                NativeArrayOptions.UninitializedMemory);

        new Job { bodies = bodies }.ScheduleParallel(query);

        state.Dependency = Physics.BuildCollisionLayer(bodies).WithSettings(settings).ScheduleParallel(out CollisionLayer layer, Allocator.Persistent, state.Dependency);
        var bcl          = new BulletCollisionLayer { layer = layer };
        latiosWorld.sceneBlackboardEntity.SetCollectionComponentAndDisposeOld(bcl);
    }

    [BurstCompile]
    partial struct Job : IJobEntity
    {
        public NativeArray<ColliderBody> bodies;

        public void Execute(Entity entity,
                            [EntityIndexInQuery] int entityInQueryIndex,
                            in WorldTransform worldTransform,
                            in Collider collider,
                            in PreviousTransform previousPosition)
        {
            CapsuleCollider capsule     = collider;
            float           tailLength  = math.distance(worldTransform.position, previousPosition.position);
            capsule.pointA              = capsule.pointB;
            capsule.pointA.z           -= math.max(tailLength, math.EPSILON);

            bodies[entityInQueryIndex] = new ColliderBody
            {
                collider  = capsule,
                entity    = entity,
                transform = worldTransform.worldTransform
            };
        }
    }
}
```

Lastly, if you just need an empty layer, you can do this:

```csharp
var settings   = new CollisionLayerSettings { worldAabb = new Aabb(-300f, 300f), worldSubdivisionsPerAxis = new int3(3, 4, 4) };
var emptyLayer = CollisionLayer.CreateEmptyCollisionLayer(settings, Allocator.Persistent);
```

## Visualizing Layers

You can visualize the elements relative to their buckets in a collision layer
using `PhysicsDebug`. This time, the method is called `DrawLayer()`, and it
offers a Fluent API for scheduling jobs.

`DrawLayer()` will draw the axis-aligned bounding boxes of each element, using a
color based on the bucket it belongs to. By default, white is reserved for the
cross-bucket. If too many of your colliders are white, you may want to consider
changing your `CollisionLayerSettings`.

You can also use `PhysicsDebug.LogBucketCountsForLayer()` to get a console log
with statistics about the collision layer. This is also a Fluent API.

## Using Collision Layers

Collision Layers can be passed into jobs as `[ReadOnly]` and queried. Many of
the Physics query operations have overloads which take a `CollisionLayer` as an
argument. For example, here’s the skeleton of a typical collide-and-slide
algorithm used in character controllers:

```csharp
[BurstCompile]
partial struct CollideAndSlideCharacterJob : IJobEntity
{
    [ReadOnly] public CollisionLayer environmentLayer;

    public void Execute(TransformAspect transform, in Collider collider, in CharacterController characterController)
    {
        var distanceRemaining = math.length(characterController.moveVector);
        var currentTransform  = transform.worldTransform;
        var moveDirection     = math.normalize(characterController.moveVector);

        float skinEpsilon = 0.001f;

        for (int iteration = 0; iteration < 32; iteration++)
        {
            if (distanceRemaining < skinEpsilon)
                break;
            var end = currentTransform.position + moveDirection * distanceRemaining;
            if (Physics.ColliderCast(in collider, in currentTransform, end, in environmentLayer, out var hitInfo, out _))
            {
                currentTransform.position += moveDirection * (hitInfo.distance - skinEpsilon);
                distanceRemaining         -= hitInfo.distance;
                if (math.dot(hitInfo.normalOnTarget, moveDirection) < -0.9f) // If the obstacle directly opposes our movement
                    break;
                // LookRotation corrects an "up" vector to be perpendicular to the "forward" vector. We cheat this to get a new moveDirection perpendicular to the normal.
                moveDirection = math.mul(quaternion.LookRotation(hitInfo.normalOnCaster, moveDirection), math.up());
            }
        }
        transform.worldPosition = currentTransform.position;
    }
}
```

There’s additionally a `FindObjects()` method that allows you to scan for all
colliders within a search region.

```csharp
[BurstCompile]
partial struct FindNearbyFoodJob : IJobEntity
{
    [ReadOnly] public CollisionLayer foodLayer;
    [ReadOnly] public ComponentLookup<FoodAmount> foodLookup;

    public void Execute(ref CharacterController characterController, in WorldTransform transform, in Collider collider)
    {
        int bestFood = 0;

        var searchRegion = new Aabb { min = transform.position - 10f, max = transform.position + 10f };
        foreach (ref readonly FindObjectsResult candidate in Physics.FindObjects(searchRegion, foodLayer))
        {
            var candidateFoodAmount = foodLookup[candidate.entity].amount;
            if (candidateFoodAmount > bestFood)
            {
                bestFood = candidateFoodAmount;
                characterController.moveTarget = candidate.body.transform.position;
            }
        }
    }
}
```

You can also iterate through all the colliders and all their data via the
`colliderBodies` property. Note that these bodies will be spatially reordered
relative to how they were created. Use the `srcIndices` property to get the
original creation index (`entityInQueryIndex` or `ColliderBody` array index).

### ColliderBody Warning

The Collider and `TransformQvvs` associated with an Entity in a `CollisionLayer`
is a **copy** captured at the `CollisionLayer` creation time, and will **not**
automatically update if you modify the `WorldTransform` or `Collider` components
on the entity.

## On to Part 3

Spatial queries on collision layers are powerful and flexible. But they come
with the limitation that if we want to use them in parallel jobs, we can only
read data from the entities in a collision layer. In the next part, we’ll
explore FindPairs, an algorithm that gives us parallel write-safe access to
pairs of entities, and is also even faster!

Continue to [Part 3](Getting%20Started%20-%20Part%203.md).
