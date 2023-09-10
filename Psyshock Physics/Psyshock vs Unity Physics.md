# Psyshock vs Unity Physics

Psyshock and Unity Physics is a contentious topic with a long history. For
almost every design decision Unity Physics makes, Psyshock ends up making the
opposite. This table breaks down the differences. And below that is the rant
that led to this divergence in the first place.

|                          | Unity Physics                                                             | Psyshock                                                                                                      |
|--------------------------|---------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Query API                | Member methods                                                            | Static methods                                                                                                |
| Collider Types           | Terrain                                                                   | More new types planned                                                                                        |
| Collider Representation  | Blob Asset                                                                | Mutable struct which for some types may contain a Blob Asset                                                  |
| Box Collider             | Convex Hull with GJK                                                      | Center and Extents with SAT                                                                                   |
| Convex Collider          | Beveled for performance                                                   | Hard-Edged for performance                                                                                    |
| Raycasts                 | Include inside hits                                                       | Ignore inside hits (use point queries instead)                                                                |
| Raycast Optimizations    | Ignore convex bevels                                                      | No bevels to ignore                                                                                           |
| Collider Casts           | Conservative Advancement with tolerance and max of 10 iterations          | Minkowski raycasts and MPR with precise results                                                               |
| Layers                   | Based on masks                                                            | Based on EntityQueries or user-provided                                                                       |
| Broadphase               | BVH with static and dynamic                                               | Multibox SAP with layer-self tests and layer vs layer tests                                                   |
| Collsion Events          | All hits stored in buffer iterated by ICollisionEventsJob single-threaded | User-invoked broadphase dispatches hits immediately to IFindPairsProcessor in parallel with pair write safety |
| Pair Data                | No guarantees, requires filtering                                         | Strong archetype guarantees                                                                                   |
| Collision World          | Owned by singleton                                                        | Fully managed by user                                                                                         |
| Children and hierarchies | Static only                                                               | Fully supported                                                                                               |
| Systems                  | Out-of-the-box systems                                                    | No out-of-the-box systems, no automatic performance cost                                                      |
| Simulator Modification   | User callbacks or intermediate systems                                    | User drives each step from anywhere                                                                           |
| Collider Scaling         | Uniform Scale                                                             | Non-uniform stretch                                                                                           |
| Integrator               | Built-in damping and gravity                                              | Utilities to aid user in writing their own                                                                    |
| Forces                   | Provide force directly                                                    | Utilities for advanced drag and buoyancy for ballistics                                                       |
| Collision Solver         | Jacobian                                                                  | Several Planned                                                                                               |
| Joints                   | Several                                                                   | Planned                                                                                                       |
| Motors                   | Several                                                                   | Planned                                                                                                       |

## The Rant

I get asked this a lot. Why? Is Unity.Physics not good enough? Is Havok’s
solution not good enough?

Trust me, when you hear my use case, you may find yourself wondering, “How the
heck is Unity.Physics supposed to be a good general-purpose design?” Spoiler
alert: It’s not. It’s very targeted at the use cases Unity is targeting.

First, let me introduce you to the decisions that make no sense to me:

-   Collider shapes exist as immutable shared data structures
-   A simulation requires all steps to be ticked rather than allowing each step
    to be ticked manually by the user
-   Simulation callbacks are single-threaded
-   Physics is very decoupled from ECS, trashing the transform hierarchy and
    using layers rather than the superior ECS query system; yet simultaneously,
    it is tightly coupled as colliders use blobs, `RigidBody` has an `Entity`
    field, and all of its authoring uses baking

Now, to understand why this is terrible for me, let me present to you a game
idea a friend of mine and I came up with during a game jam:

The game was an RTS mixed with a TPS. You played as a witch or wizard defending
a magical forest from a military force that was seeking to harvest the forest’s
energy. You and your tower defense animals shot zany spells and spastic
projectiles at the enemy forces. You as the player needed to manage resources
and protect your land while simultaneously moving on foot to capture the enemy
military bases. Oh, and the spells could have all sorts of weird effects
including scaling things.

So let’s examine how Unity.Physics stacks up:

-   First and foremost, we need a Physics solution. The TPS mechanics of
    shooting and interacting with the world require the precision of real
    physics and not some weak position checks.
-   We need a mostly static environment for our characters and projectiles to
    traverse. Unity.Physics handles that perfectly well.
-   We need colliders on animated characters. Actually, we are still good here
    too. Even though Unity.Physics trashes the hierarchy, it doesn’t trash the
    `LinkedEntityGroup`, which means we can still instantiate a prefab with all
    of the individual colliders and reference them to drive them with bones.
-   We need morphing collider shapes. We only needed uniform scaling in this
    case, so Unity Physics 1.0 is sufficient. However, if we needed non-uniform
    scaling to sync with squishy animations, then we would have to allocate and
    deallocate blob assets every frame which is pretty awful.
-   Most of our game logic is going to be detecting if two colliders
    intersected. We want to know if our friendly projectiles hit the enemies, if
    the enemy fire hits us, if any projectile hits terrain, if two different
    spells hit each other, ect. Different collisions require different logical
    responses. At first, Unity.Physics looks like it should handle this fine,
    but this is actually where it falls flat on its face the hardest. Allow me
    to explain…

The first issue is we need to tell Unity what collides with what. Right now we
can characterize that based on EntityQueries, because we are using Tags and
unique component types for all of our other logic. But for Unity.Physics we need
to encode all of that into a layer mask system. While annoying, it is doable.

After having told Unity what collides with what, it generates a `NativeStream`
to be processed in an `ITriggerEventsJob` giving us all of our collisions. That
seems nice, except all of our resulting pairs are mixed together like a jar of
jelly beans. Consequently each unique event handler needs to filter through the
results and pick out the ones it cares for. That’s a lot of iterating through
the events. We can use the new `EntityQueryMask` to speed this up, but still, it
isn’t great. Instead, we might decide to bring our iteration count back down by
having one job do all the filtering for all the different handlers to react to
by creating a bunch of smaller collections of pairs. This works, but now the
global filterer needs to know about every kind of pair interaction. You can try
to generalize this, but it is just clunky. And also, this mega-filter algorithm
has to run single-threaded. So this is a pretty bad bottleneck and
simultaneously is tangling all our code together in stupid ways.

What, you thought that was bad? It only gets worse.

Let’s suppose that we decide to drive our moving objects using physics, but we
run into issues with the contacts and solver behavior between the military
vehicles and the character controllers. We want to implement our own solver for
just these interactions. Well guess what? Remember that song and dance we had to
play to sort all of the collision events to the proper handlers from a single
thread? We have to do the same thing again for the contacts, the jacobians, and
pretty much any stage of the simulation. This time we only care about a small
fraction of these events, but we still have to sift through all of them. That’s
a pretty lame performance tax. All of the Unity.Physics callbacks suffer from
this.

Unity.Physics feels like a simulator, not a real physics engine for gameplay.
Here’s some common gameplay examples that humiliate Unity.Physics as a
general-purpose physics engine:

What if I wanted to drive the colliders to spin in a circle around a character
but still be individually destroy-able (think boss shields or Mario Kart’s
triple shells in the newer games)? I have to set up everything with joints
rather than rely on the Transform System with a rotating parent.

What if I wanted to destroy an otherwise static rock from an explosive? Well now
the entire static world needs to be rebuilt.

What if I am trying to get sparks to bounce off a flash mob of vocaloids which
need a dynamically updated mesh collider every frame? That cooker is expensively
slow!

What if I am building a platformer and want to mimic the expanding and
contracting mushrooms from the New Super Mario Bros games? Do I create a
collider for every frame for those too? Or do I prebake the colliders for every
step in the cycle?

What if I am procedurally generating a world and want to apply scale variations
to the instantiated prefabs? More slow cooking.

What if I want to make an active ragdoll with rigid bodies for bones on an
animated character? Unity Physics will break apart the entire hierarchy which
will probably break animations.

More often than not, I find myself fighting with Unity.Physics (and
Havok.Physics) thinking backwards about my problems trying to not pay the cost
for things I don’t need.

So I presented my concerns on the forums, and then decided that if I wanted it
done right, I was going to have to do it myself. I’m definitely not there yet,
but I am comfortable and confident in the direction I am headed.
