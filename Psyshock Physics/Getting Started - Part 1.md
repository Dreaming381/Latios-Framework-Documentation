# Getting Started with Psyshock Physics – Part 1

Psyshock is an ever-growing kit of tools and algorithms for spatial queries and
physics simulations. You have likely never encountered a physics package like
Psyshock, and consequently, the learning curve may be a little steep. The basic
premise is that you have near complete control over your data as well as what
happens and when. There’s no engine. Just engine parts for you to assemble your
own.

While the learning curve is steep, it will be easier than writing your own
physics engine from scratch though. And if you considered writing your own
physics engine, you may be able to save time and debugging using Psyshock’s
low-level APIs instead.

This Getting Started series is broken up into multiple parts. This first part
will introduce the basics of colliders. Colliders are the basic building blocks
of Psyshock. They represent far more than just collision. You can use them for
triggers, sensors, volumes, and even as debug gizmos.

## Authoring

Currently Psyshock uses the classical Physx components for authoring
[colliders](Colliders.md). Attach a Sphere Collider, Capsule Collider, Box
Collider, or Mesh Collider to the GameObject you wish to convert.

There’s one exception. I added support for Compound Colliders and this uses the
new component found at *Latios -\> Physics (Psyshock) -\> Custom Collider*. But
you can also create them by attaching more than one Physx collider component to
a Game Object.

## Only Your Code Matters

Aside from authoring, nothing in Psyshock happens without an explicit call from
you. No collision detection happens unless your code asks for it. No motion or
gravity happens either.

Unlike with Unity Physics, you are in complete control of the simulation from
start to finish. Psyshock simply provides a toolbox of powerful tools to help
you along the way. You get all the benefits of a custom physics engine for your
game (especially performance) without having to know all the math and algorithm
details.

This is great power that comes with some responsibility. Using the tools wrong
will almost certainly result in poor performance. If you plan to dive deep into
Psyshock, have the profiler nearby. You are always welcome to send your project
to me for suggestions on how to improve performance if you find physics to be
your culprit.

## Colliders in code

```csharp
//How to create Collider

//Step 1: Create a sphere collider
var sphere = new SphereCollider(float3.zero, 1f);

//Step 2: Assign it to a Collider (union of all collider types)
Collider collider = sphere;

//Step 3: Attach the Collider to an entity
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

You don’t always have to use `Collider` as a component either. You can make any
specific collider type or even the union `Collider` type a field in any of your
components or dynamic buffers.

```csharp
struct AttackData : IComponentData
{
    public SphereCollider   weakHitbox;
    public CompoundCollider strongHitbox;
    public Collider         specialHitbox;
}
```

## Other Essential Types

Psyshock contains several other types you may need to provide for its various
APIs.

-   Ray – A ray is composed of start and end points, and is used for raycasts
-   Aabb – An axis-aligned bounding box, often used for cheap conservative tests
    to filter out colliders nowhere near each other.

## Visualizing Colliders

The static class `PhysicsDebug` provides various methods all named
`DrawCollider()` which will draw a rough wireframe of the collider’s shape in
the scene view. For rounded colliders, there is an optional parameter to specify
the density of segments.

## Simple Queries

There are several static query methods that operate on a single collider or a
pair of colliders.

-   Shape
    -   Physics. AabbFrom
-   Spatial
    -   Physics.Raycast
    -   Physics.DistanceBetween
    -   Physics.ColliderCast
-   Simulation
    -   UnitySim.ContactsBetween

*Warning: Unlike Unity Physics,* `Raycast()` *and* `ColliderCast()` *do not
report inside hits or “zero-distance” hits. Use a* `DistanceBetween()` *query at
the start point to get inside hit info.*

Example usage of `DistanceBetween()` for a bullet-hell:

```csharp
foreach ((var playerData, var playerHurtbox, var playerTransform) in 
    SystemAPI.Query<RefRW<PlayerData>, Collider, WorldTransform>())
{
    foreach ((var bulletHitbox, var bulletTransform) in SystemAPI.Query<Collider, WorldTransform>())
    {
        // DistanceBetween() returns true if the found distance is less than the maximum distance. 
        // Intersections will have negative distances, so a maximum distance of 0f will only return true if the colliders intersect.
        if (Physics.DistanceBetween(in playerHurtbox, in playerTransform.worldTransform, in bulletHitbox, in bulletTransform.worldTransform, 0f, out _))
        {
            playerData.ValueRW.isAlive = false;
            break;
        }
    }
}
```

## Simple Modifiers

There are also APIs that can modify data. Some of these create copies, while
others are able to modify the object in-place.

-   Physics.ScaleStretchCollider
-   Physics.CombineAabb
-   Physics.TransformAabb

*Note: For queries that accept both a* `Collider` *and a* `TransformQvvs`*, you
do NOT need to pre-apply the scale and stretch. The most likely scenario where
you would need* `ScaleStretchCollider()` *would be when baking a collider to a
custom component or additional entity.*

## On to Part 2

Colliders and basic queries are already a powerful and practical arsenal in
DOTS. And maybe, that’s all your game needs. But most likely, you also wish to
query entire collections of colliders scattered across your game world. In the
next part, we’ll explore *collision layers*, which can do just that.

[Continue to Part 2](Getting%20Started%20-%20Part%202.md)
