# Choosing the Right Transform System

The Latios Framework offers two choices for how transforms may work in your
project. Transforms affect nearly every aspect of a project. Picking the correct
one is important.

-   QVVS Transforms
-   Unity Transforms

## QVVS Transforms

QVVS Transforms is an always up-to-date transform system built upon the
`TransformQvvs` type. It uses a `WorldTransform` component for expressing the
world-space transform of every entity. Additionally, root entities of
hierarchies with children may have a `DynamicBuffer<EntityInHierarchy>`, while
children entities have a `RootReference`.

The `DynamicBuffer<EntityInHierarchy>` is stored outside ECS chunks. Therefore,
solo entities require 48 bytes of transform data, root entities require 64
bytes, and child entities require 60 bytes.

QVVS Transforms is always up-to-date, which means changes in parent transforms
are immediately propagated to children, and the hierarchy is always fully
available to navigate. There is no dedicated ECS system for synchronizing the
hierarchy.

When working with QVVS Transforms, the general philosophy is if you want to read
the `WorldTransform`, just read the component directly. And if you want to write
transforms or read local space, use `TransformAspect` which abstracts hierarchy
details.

QVVS Transforms is the default transform system in the Latios Framework. It
supports all features, including Motion History, Inheritance Flags, and
`GameObjectEntity`.

## Unity Transforms

Unity Transforms is the solution provided by default ECS. It uses a non-cached
matrix-centric workflow. There is a `LocalTransform` component and a
`LocalToWorld` component present on every moving entity.

The `DynamicBuffer<Child>` has an in-chunk capacity of 0, requiring 16 bytes of
chunk space. `LocalToWorld` is also 16 bytes larger than a QVVS because it uses
a `float4x4` instead of a `float3x4`. The systems may be plagued with
determinism flaws with regards to its hierarchy. Entity in chunk order and
change filters are not deterministic when changing parents at runtime. In
addition, because it is matrix-based, additional computation is required for
accessing world-space transform data for Psyshock, Myri, and rendering with
Entities Graphics or Kinemation.

Despite these flaws, Unity Transforms offers the greatest amount of
compatibility with other assets and packages. It also provides better
performance when scaling entities on custom pivots.

Excluding `Parent` and `Child` components, root and child entities each require
96 bytes.

When working with Unity Transforms, `LocalToWorld` may be stale even for root
entities, but is generally the component to read from for world-space
transforms. When writing to transforms, always write to the `LocalTransform`
component.

The hierarchy is not always up-to-date, and is explicitly synchronized by the
`TransformSystemGroup` each frame.

Unity Transforms must be enabled with the LATIOS_TRANSFORMS_UNITY scripting
define.

## How to Decide

If you are adding the framework to an existing project, and aren’t near your
breaking point where you want to rewrite your entire codebase, then definitely
use Unity Transforms. Switching to QVVS Transforms is a massive undertaking, and
probably isn’t worth it.

The next factor to consider is which Unity packages you want to use. If you want
to use Unity Physics, the Character Controller, or the Vehicle Controller
packages, then you will want to use Unity Transforms, as these packages are not
compatible with QVVS Transforms.

If your game is going to rely heavily on complex rigid body dynamics, then you
probably want to stick with Unity Physics. While Psyshock has rigid body physics
support, it is not as production-ready as Unity Physics, and lacks some
features. It is good enough for some simple rigid body mechanics. If you are
only considering Unity Physics for its spatial queries, you might want to
consider Psyshock instead (because Psyshock is really good at that).

If you need complex rigid body dynamics, but don’t want to use Unity Physics
because it can’t do what you want or you just don’t like it, then you might
consider QVVS Transforms more heavily. Psyshock is much easier to extend if you
want to build a completely custom solution yourself. Or you might try to
integrate another physics engine such as Jolt.

For character controllers, you might choose Unity Transforms so that you can use
the Character Controller package with Unity Physics. However, Psyshock has a
stronger spatial query engine that you may prefer if you plan to write a custom
character controller.

NetCode is another package you want to consider heavily. And it largely depends
on how you plan to split up your entities. Some NetCode projects will use
dedicated networking entities separate from the simulation. Systems will copy
data to and from these entities before and after simulation. If you plan to use
NetCode this way, QVVS Transforms work well.

However, the more common convention is to use NetCode ghost entities directly in
the simulation. If you plan to do this, Unity Transforms will probably provide a
better experience.

If you got this far and QVVS Transforms is still a contender, then it really
comes down to which you prefer. Latios Framework Add-Ons tend to favor QVVS
Transforms, while the broader Unity ecosystem favors Unity Transforms. QVVS
Transforms are developed and maintained by an open-source developer who doesn’t
like Unity Transforms. Unity Transforms are developed and maintained by Unity.

If you still can’t decide, try doing a weekend game jam with each, and see if
you prefer one over the other.
