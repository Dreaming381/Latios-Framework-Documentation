# Psyshock Physics

Psyshock is a build-your-own-physics-engine toolkit with a powerful spatial
query API and various simulation building blocks to provide complete control
over all things spatial and physics.

While the API is lower-level than a typical physics engine, Psyshock allows you
to sculpt a spatial query and/or physics engine suited for your project with
better performance and gameplay ergonomics than a traditional physics engine can
provide.

Check out the [Getting Started](Getting%20Started%20-%20Part%201.md) series!

## Why?

Unity Physics and Psyshock have very different design goals. Unity Physics is
aimed towards designers and out-of-the-box simulation. Psyshock is aimed towards
gameplay programmers and extreme flexibility. If both these solutions were
pizzas, Unity Physics would be a supreme pizza, whereas Psyshock would be a
choose-your-own-toppings pizza, with more and more toppings being stocked in
each release.

If Unity Physics can’t do what you want, or does more than you want and tanks
your performance, Psyshock may provide a faster route to success!

For a more detailed comparison including the rant which used to be at the top of
this page, look [here](Psyshock%20vs%20Unity%20Physics.md).

## Features

### Mutable Colliders

Instead of making the [Collider](Colliders.md) component a pointer to an
immutable asset, Psyshock puts the mutable properties directly inside the
Collider component. Don’t worry. You don’t have to query for every
`SphereCollider`, `CapsuleCollider`, `BoxCollider`, `TriangleCollider`, ect.
Psyshock uses a union to pack the different types together. The primitive
colliders fit entirely in the component, which makes sense because those are the
colliders you are going to want to mutate every which way possible anyways. The
more complex colliders may expose some readily tweakable parameters (like scale)
but also rely on Blobs for stuff that wouldn’t otherwise fit in a copyable
struct.

One might argue that copying large collider components (it is the same size as
`WorldTransform`) is quite a bit more expensive than copying pointers. But given
how many other performance-hogging decisions Unity.Physics makes, I will take
this trade any day.

As a bonus, Psyshock colliders are constructable and copyable on the stack. That
makes them a lot more pleasant to work with.

### Transform Hierarchy Support

The physics algorithms are able to capture world-space transforms from child
entities in hierarchies. Additionally, when using QVVS Transforms and the
`UnitySim` APIs, you can use `TransformAspect` to commit rigid body world-space
transforms to child entities.

QVVS Transform features like scale and stretch are also supported and work
automatically. For shapes which don’t have well-defined stretch algorithms, you
can choose from one of several approximation methods on a per-collider basis.

### Infinite Layers

Do you want layers defined by entities matching queries rather than bitfields?
You’ve come to the right place!

Fun fact, Unity.Physics default `BuildPhysicsWorldSystem` doesn’t have one
broadphase structure. It has two. One is for statics, and the other is for
dynamics. It performs a bipartite check between the two. So then if bipartite
checks are cheaper than building the static world broadphase every frame, is
there a reason why we don’t just build a broadphase structure for each group of
colliders we care about?

That was the question I asked myself, and then I said, “No, there is not!”

Instead of building two large broadphase structures with layer masks and then
untangling the results, you can build a [CollisionLayer](Collision%20Layers.md)
per unique `EntityQuery` (or from arbitrary data from a job). Then you can ask
for all collisions within a single layer or for the collisions between layers.
You can also query against each layer separately. This gives you fine-grain
control over which sets of colliders test against each other, with very little
overhead. And you can explicitly specify the interaction behavior using…

### FindPairs – A Multibox Broadphase

FindPairs lets you find candidate overlaps blazingly fast, and process pairs
with your own logic with some powerful guarantees.

In Psyshock, you call `Physics.FindPairs()` and first pass in one or two
collision layers as arguments. Then you pass in a generic argument implementing
`IFindPairsProcessor`. This is the struct that will hold your NativeContainers
and other resources to handle the results.

When scheduling with two layers, each pair will always result in the first pair
entity coming from the first layer, and the second pair entity coming from the
second layer. You’ll always know exactly what you’re getting!

`Physics.FindPairs` can be scheduled as a parallel job, and when you do, the
algorithm guarantees that both entities passed into the
`IFindPairsProcessor.Execute()` are **thread-safe writable** to any of their
components!

Do you know how many people have asked about how to write in parallel to nearby
enemies that should be damaged by an attack or some similar problem? You don’t
have to record events to some separate container and attempt to sort them by
entity. You can simply write what you need right in the processor.

It’s thread-safe. It’s parallel. It’s magic!

Not entirely…

You can still shoot yourself in the foot by trying to access an entity that
isn’t one of the found pair’s entities. To stay safe, you can use
`PhysicsComponentLookup`, `PhysicsBufferLookup`, and `PhysicsAspectLookup` (all
implicitly assignable from the Entities package counterparts) when you need
write access to entities found by FindPairs.

The only other thing you must watch for, is that **no entity can appear twice in
a participating** `CollisionLayer`**!** There is a check in place for this when
safety checks are enabled. If this rule is a problem for you, use
`ScheduleSingle()` instead.

Another awesome thing about FindPairs is that it is a broadphase. It reports
AABB intersection pairs, not true collider pairs. Why is that a good thing? It
means you get to choose what algorithm to follow up with.

-   Need all the contact manifold info?
    -   Cool!
-   Just need the penetration and normal to separate them?
    -   You can do that too.
-   Do you want a cheap intersection check to keep the cost down?
    -   That’s a possibility as well.
-   Or maybe your triggers involve fast-moving entities and you need a collider
    cast for continuous collision detection?
    -   Consider it solved.
-   Do you want to check if the entities have some other component values before
    you do the expensive intersection checks?
    -   Also viable.
-   How about firing a bunch of raycasts between the two colliders to build a
    bunch of buoyancy force vectors?
    -   Actually, yes. It is pretty similar to an `IJobParallelFor` in that
        regard.

And remember, you can use a different `IFindPairsProcessor` for each FindPairs
call.

### CollisionWorld – Injecting ECS into CollisionLayers

Have you ever wanted to do a raycast, and specify `IComponentData` types that
the hit entities must have to be accepted?

`CollisionWorld` is a special type of `CollisionLayer` with extra ECS metadata
baked in to do exactly this! But it doesn’t just filter results. It uses the
specified components to actively reduce the set of AABBs it needs to search. To
my knowledge, no other ECS physics engine can do this!

### Static Queries

No OOP-like API here. All the queries are static methods in the static Physics
class. Combined with the stack-creatable colliders, these methods can be quite
useful for “What if?” logic.

The common APIs that deal with colliders are in the static `Physics` class. For
more niche queries, the `QuickTests` class might just have what you are looking
for.

### No Hacks

Did you know that casting an infinitely small sphere collider, like one with 0
radius, is equivalent to a Raycast? So let’s say you have a 1x1x1 box collider
with a bevel radius *r* at the origin and you cast a ray and a sphere starting
at (-10, -10, -10) towards the origin. Theoretically, you should get the same
hit point in world space. Yet if you compare the results of these operations
using Unity.Physics, you will get two different hit points in world space. In
fact, the distance between those two points is expressed by this formula: *r(√3
– 1)*

This isn’t the only hack in Unity.Physics. Any `CastCollider` function will
translate the casted collider along the ray and then check if it is “close
enough”. And if it doesn’t find something after 10 tries, it just gives up and
calls it a miss.

These are hacks, designed to provide slight performance benefits for most people
who don’t care about them. Yet these hacks can also be the source of
difficult-to-understand bugs and long weeks of “why is my object placement math
code wrong?”

Psyshock is hack-free. Queries generate consistent results with as much
precision as the 32-bit floating point hardware allows. And yes, this leads to
some very interesting `ColliderCast` algorithms.

### Immediate Mode Design

Nearly the entirety of Psyshock is composed of data types, data structures, and
stateless static methods. There’s no internal state. There’s no
“StepTheSimulation” method (and if there ever becomes one, it will just be a
convenience method using public API). There are no systems (other than baking).

Even the debug tools are static!

Unity.Physics first and foremost tries to be an out-of-the-box solution with all
flexibility features being baked into the simulation logic. However, this often
comes at the cost of performance. For example, in Unity.Physics, triggers are
determined by simulating a contact impulse, resulting in contacts being
generated in situations where all you care about is whether or not a body
entered some trigger volume.

In constrast, Psyshock tries to be a “Build your own solution” framework and is
being designed inside-out. Over time, it will eventually achieve an
out-of-the-box status, but that is not the focus. The focus is flexibility with
the luxury of drop-in optimized and accurate algorithms and authoring tools.

### Invent Your Own Laws

For most physics engines, if you need something extremely custom, you either
have to put in the hooks in the right places to trick the engine into doing what
you want, or even worse, modify the code directly. In Psyshock, you own the
engine. It is up to you whether or not you want to use the physical rules
provided or make up your own. Do you want every object to experience its own
rate of time? Such a concept would usually require a custom physics engine. But
with Psyshock, this can be achieved with far less effort.

Psyshock doesn’t come with an out-of-the-box constraint solver, but rather
provides the constraint-solving algorithms and a powerful `PairStream` data
structure that makes it simple and safe for you to build your own solver. This
also means you can define your own constraints, such as custom rules for
character controllers or magical forces.

## Known Issues

-   Compound and TriMesh colliders do not use the most optimal runtime data
    structures due to the prohibitive baking costs associated
-   Compound Colliders do not support embedded Triangle, Convex, or TriMesh
    Colliders
-   This Readme has too many words and not enough pictures.

## Near-Term Roadmap

-   More QuickTests
-   More Character Controller Utilities
-   Motor Constraints
-   Terrain Collider Baking
-   Persistent Pair State
-   FindPairs improvements
    -   Aabb-only layers
    -   Mismatched layers support
    -   Distance inflation
    -   Optimizations
-   FindObjects improvements
    -   “Any” early-outs
    -   Optimizations
-   More Collider Shapes
    -   Quad, RoundedBox, Cone, Cylinder
    -   Marching Cubes, Convex Compound (V-HCAD)
-   Simplified Overlap Queries
-   More Force Equations
-   More Constraint Formulas and Solvers
-   General Optimized Contact Generation
-   Authoring Improvements
    -   Autofitting
    -   Stretch mode baking options
-   More Debug Tools
-   Debug Gizmo Integration Hooks
    -   If you are an asset developer interested in this, please reach out to me
