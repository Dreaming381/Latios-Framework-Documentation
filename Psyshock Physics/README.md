# Psyshock Physics

Well… really more like a spatial query engine at this point, but whatever.

Check out the [Getting Started](Getting%20Started.md) page!

## Why?

Unity Physics and Psyshock have very different design goals. Unity Physics is
aimed towards designers and out-of-the-box simulation. Psyshock is aimed towards
gameplay programmers and extreme flexibility. If both these solutions were
pizzas, Unity Physics would be a supreme pizza, whereas Psyshock would be a
choose-your-own-toppings pizza…

…with a bunch of toppings out of stock.

If Unity Physics can’t do what you want, or does more than you want and tanks
your performance, Psyshock may provide a faster route to success!

For a more detailed comparison including the rant which used to be at the top of
this page, look [here](Psyshock%20vs%20Unity%20Physics.md).

## Features

*Disclaimer: Some of the details of the features listed in this section are not
available yet but are planned for a near future release and have been heavily
accounted for in the design. If a particular feature shows promise in solving a
use case you are currently struggling with, please let me know so that I can
prioritize it. Solving other people’s problems seems to give me some
productivity buff. I don’t know. It’s probably some kind of fairy magic.*

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
`LocalToWorld`) is quite a bit more expensive than copying pointers. But given
how many other performance-hogging decisions Unity.Physics makes, I will take
this trade any day.

As a bonus, Psyshock colliders are constructable and copyable on the stack. That
makes them a lot more pleasant to work with.

### Transform Hierarchy Support

The physics algorithms apply a local to world space conversion when capturing
data from entities. They will also support world to local space conversion on
simulation write back when simulation is supported.

Other transform features like scale and stretch are also supported and work
automatically. For shapes which don’t have well-defined stretch algorithms, you
can choose from one of several approximation methods.

### Infinite Layers

So fun fact, Unity.Physics default `BuildPhysicsWorldSystem` doesn’t have one
broadphase structure. It has two. One is for statics, and the other is for
dynamics. It performs a Bipartite check between the two. So if Bipartite checks
are cheaper than building the static world broadphase every frame, is there a
reason why we don’t just build a broadphase structure for each group of
colliders we care about?

That was the question I asked myself, and then I said, “No, there is not!”

So instead of building two large broadphase structures with layer masks and then
untangling the results, you can build a [CollisionLayer](Collision%20Layers.md)
per unique `EntityQuery` (or from arbitrary data from a job). Then you can ask
for all collisions within a single layer or for the collisions between layers.
You can also query against each layer separately.

### FindPairs – A Multibox Broadphase

What is so good about a multibox broadphase? Well for one, there’s no
single-threaded initial step, which is one of Unity.Physics bottlenecks. But
second, it’s a cheap pseudo-islanding solution.

And I know what you are wondering. How is that useful?

In Psyshock, you call `Physics.FindPairs()` and pass in one or two collision
layers as arguments. However, there is an additional generic argument requesting
an `IFindPairsProcessor`. This is the struct that will have your
NativeContainers and stuff to handle the results.

`Physics.FindPairs` can be scheduled as a parallel job, and when you do, the
algorithm guarantees that both entities passed into the
`IFindPairsProcessor.Execute()` are **thread-safe writable** to any of their
components!

Do you know how many people have asked about how to write in parallel to nearby
enemies that should be damaged by an attack or some similar problem? Well here
you just have to create a trigger collider with a large radius and run its layer
against the layer with your enemies. Damage will stack as expected. Or you can
set a bool that says future interactions with that enemy can’t happen anymore.
And at the same time, you can do stuff on the attack trigger too, like record
all the enemies it touched into a dynamic buffer.

It’s thread-safe. It’s parallel. It’s magic!

Not entirely…

You can still shoot yourself in the foot by trying to access an entity that
isn’t one of the pair’s entities. That includes trying to access a parent
entity, which is a common use case. You’ll have to be smart about how you go
about that problem, or just avoid it by not using
`[NativeDisableParallelForRestriction]`. There’s a `PhysicsComponentLookup` type
that will help you follow the rules when writing to components.

But even more importantly, you must ensure that **no entity can appear twice in
a participating CollisionLayer!** There is a check in place for this when safety
checks are enabled. If this rule is a problem for you, use `ScheduleSingle()`
instead.

There’s still one more awesome thing about FindPairs. It is a broadphase. It
reports AABB intersection pairs, not true collider pairs. Why is that a good
thing? It means you get to choose what algorithm to follow up with.

-   Need all the contact manifold info?
    -   Cool!
-   Just need the penetration and normal to separate them?
    -   You can do that too.
-   Do you want a cheap intersection check to keep the cost down?
    -   That’s a possibility as well.
-   Do you want to check if the entities have some other component values before
    you do the expensive intersection checks?
    -   Also viable.
-   How about firing a bunch of raycasts between the two colliders to build a
    bunch of buoyancy force vectors?
    -   Actually, yes. It is pretty similar to an `IJobParallelFor` in that
        regard.

And remember, you can use a different `IFindPairsProcessor` for each FindPairs
call.

### Static Queries

No OOP-like API here. All the queries are static methods in the static Physics
class. Combined with the stack-creatable colliders, these methods can be quite
useful for “What if?” logic.

Also, you might see some more non-traditional queries pop up at some point if I
find myself wanting parabolic and smoothstep trajectories again.

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

Unity.Physics first and foremost tries to be an out-of-the-box solution and then
slowly is working on exposing flexibility.

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
with Psyshock, this can be achieved with little effort.

## Known Issues

-   This release is missing quite a few collider shapes, queries, and simulation
    features.
-   Compound Colliders use linear brute force algorithms and lack an underlying
    acceleration structure. Try to keep the count of primitive colliders down to
    a reasonable amount.
-   Authoring is weak right now. That stuff takes me a while to get working.
-   This Readme has too many words and not enough pictures.

## Near-Term Roadmap

-   More Character Controller Utilities
-   FindPairs improvements
    -   Aabb-only layers
    -   Pair filter caching
    -   Mismatched layers support
    -   CollisionLayers fully deferrable
    -   Bucket begin/end callbacks
    -   Distance inflation
    -   Optimizations
-   FindObjects improvements
    -   “Any” early-outs
    -   Optimizations
-   More Collider Shapes
    -   Quad, RoundedBox, Cone, Cylinder
    -   Terrain, Static Mesh, Convex Compound (V-HCAD)
-   Simplified Overlap Queries
-   Manifold Generation
-   More Force Equations
-   Authoring Improvements
    -   Autofitting
    -   Stretch mode baking options
-   More Debug Tools
-   Debug Gizmo Integration Hooks
    -   If you are an asset developer interested in this, please reach out to me

## Not-So-Near-Term

-   Simulations (The first piece of this is Manifold Generation)
-   Spatial hash broadphase
