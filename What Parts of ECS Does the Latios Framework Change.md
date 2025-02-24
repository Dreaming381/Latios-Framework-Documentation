# What Parts of ECS Does the Latios Framework Change?

This is a common question that circulates around. And unless you talk to someone
who uses the framework in the [Latios Framework
Discord](https://discord.gg/DHraGRkA4n), you may hear misinformation about it.

1.  **No** feature requires a change in workflow, except to the functionality it
    directly pertains to.
2.  The Latios Framework does **not** change anything unless you ask it to.
3.  Many “alternative” features **co-exist** with existing ECS features (i.e.
    singletons and blackboard entities, or Unity Physics and Psyshock)

Every change must be opted into. If you would rather stick to the classical ECS
way of using singletons and attributes to order your systems and using Unity
Physics, you can do all that and still use some of the powerful features and
optimizations included in the Latios Framework.

This page describes all of the various opt-ins that result in explicit changes
in the ECS ecosystem, as well as what other benefits are unlocked with each
opt-in.

## I’m a Beginner. Should I Start Using This?

If you are completely brand-new, you will want to familiarize yourself with the
absolute basics first, before jumping in. However, the point where you should
consider starting to explore the Latios Framework is much sooner than most will
tell you.

My suggestion would be to first learn to use jobs and Burst. Then learn to
create, move, and destroy entities. Then learn to write your own bakers and
manipulate your entities in jobs such as `IJobEntity`. And learn to use things
like `NativeHashMap` and `ComponentLookup` to express relationship operations.

If you are at that level, you are well-equipped to start leveraging the Latios
Framework. Don’t hesitate to ask questions on our discord to keep you moving
forward.

With that said, if you are the kind of person to jump off the deep end right
away, you are definitely not the first within the community. Some of our discord
members have proven it to be a viable approach.

## Changes on Installation

**Workflow-wise? Nothing!**

Okay, you’ll find that you need to add a couple of scripting defines to get it
to compile, with one being ENTITY_STORE_V1, which might be a little disruptive.
This is a stopgap to ensure full per-hardware determinism, something that the
framework guarantees.

You will probably want to add the scripting define LATIOS_TRANSFORMS_UNITY if
you decide to only go this far.

There will be three hooks added to ECS from installation. One is during the
setup of baking. One is during the creation of the Editor World. And one is
during a `ComponentSystemGroup` update. All these hooks do are check for
specific Latios Framework features being used (none are yet) and then return
back to the normal Unity way of doing things.

Nearly all Latios Framework systems use the `[DisableAutoCreation]` attribute,
so they won’t be around. The exceptions are some baking systems and a
`ProcessAfterLoad` system for Unika.

Features made available in this state include:

-   Smart Blobber and Smart Bakers
-   `IBaker` extension methods for interfaces
-   Baking system `WorldUpdateAllocator` memory leak fix
-   `BootstrapTools` functionality
-   Math and `Rng` Extensions and utilities
-   Collections and Custom Command Buffers (user-invoked playback only)
-   `EntityDataCopyKit` (and its underlying `EntityManager` extension methods)
-   Fluent Queries
-   `EntityWith<>` and `EntityWithBuffer<>`
-   `DynamicHashMap`
-   `ThreadStackAllocator`
-   `TempQuery`
-   `ComponentBroker`
-   `TypePack`, `Bits`, and other extensions
-   The `TransformQvvs` type and operations
-   All static runtime methods of Psyshock (Psyshock at runtime is system-free)
-   NetCode Bootstrap and utility APIs (when NetCode package is installed)
-   Kinemation’s `GraphicsUnmanaged` and `GraphicsBufferUnmanaged` APIs
-   Unika, with the exception of not having the default entity remap systems
-   Various other extension methods

It is very rare for people to only go this far. But it might make sense if you
are solely looking to benefit from using both Psyshock and Unity Physics in the
same project, or if you just want Unika.

## Changes When Adding the Bootstrap Unity Transforms – Injection Workflow

This is a popular way to use the framework. It preserves compatibility with the
Unity ecosystem, while still offering many of the benefits Kinemation has to
offer (plus modules like Myri and Calligraphics).

Adding the bootstrap is what activates the hooks and turns on many of the core
features of the framework. However, these core features do not break any
existing workflows. Here’s what they do:

-   Provide a bootstrap interface for customizing the editor world
-   Provide a bootstrap interface for customizing the bakers and baking worlds
-   Apply pre- and post-processing of updated systems in `ComponentSystemGroups`
    which only affects systems that use specific Latios Framework features

That last bullet point will cause the Latios Framework to show up in all system
update callstacks. But besides that, this change preserves the existing ECS
workflow and sequence of operations 100% and shouldn’t affect you in any way.

However, the bootstrap itself breaks one particular thing in ECS. You cannot use
`[CreateBefore]` on a custom system referencing a Unity system. Unity systems
are always created first, then framework systems, and then your systems.

This is done to allow you to specify in the bootstrap which framework features
you want added to the world via installers, and then to safely inject your own
systems into the `ComponentSystemGroups` those features create.

An exception exists for systems that update in `DefaultVariantSystemGroup` in a
NetCode project.

Features made available with this bootstrap include:

-   Blackboard Entities
-   Collection Components and Managed Struct Components
-   `LatiosWorldUnmanaged.syncPoint` (what can play back custom command buffers)
-   Auto-Destroy Expirables
-   `ISystemShouldUpdate` callbacks and hierarchical system culling (skip
    updates based on something other than emptiness of `EntityQueries`)

Features installed by default in this bootstrap:

-   Myri
-   Kinemation
-   Calligraphics
-   Unika default entity remap systems

### Myri

Myri makes no modifications to existing workflows.

Features made available include:

-   Support for thousands of audio source, both one-shot and looping
-   Spatialization of sources at the listener
-   Automatic voice combining

### Kinemation

Kinemation makes two big workflow changes. It changes how Mesh Renderers are
baked. And it changes how Skinned Mesh Renderers function.

Let’s suppose you have a Mesh Renderer with 5 materials:

-   OpaqueA
-   OpaqueB
-   TransparentC
-   OpaqueD
-   TransparentE

Entities Graphics will by default bake the entity like this:

-   Baked Entity
    -   Additional Entity – OpaqueA
    -   Additional Entity – OpaqueB
    -   Additional depth-sorted Entity – TransparentC
    -   Additional Entity – OpaqueD
    -   Additional depth-sorted Entity – TransparentE

Or, with submesh sharing defined, it might bake it like this:

-   Baked depth-sorted Entity – OpaqueA, OpaqueB, TransparentC, OpaqueD,
    TransparentE

The former is a lot of entities, and the latter hurts batching of opaque
materials, as each material will receive its own draw call. Kinemation does this
instead:

-   Baked Entity – OpaqueA, OpaqueB, OpaqueD
    -   Additional depth-sorted Entity – TransparentC, TransparentE

Here, the opaque materials will be drawn using instancing, while the transparent
materials will receive their own draw calls.

If you need a different layout, Kinemation allows you to override baking of
individual Mesh Renderers. You can also use this API to bake procedural meshes.

Skinned Mesh Renderers function very differently from Entities Graphics. They
use a different blend shape buffer type (that maintains history). They don’t use
skin matrices at all (these are computed on the GPU). And they are dynamically
parented to the skeleton entity at runtime.

But regarding material properties, they use the same scheme as Mesh Renderers.
You can set up override material properties. And you can use runtime meshes and
materials via `MaterialMeshInfo`.

While Kinemation swaps out many of Entities Graphics systems, aside from skinned
meshes and LODs, these changes only expose new features, enhance performance,
and fix lingering bugs.

As for LODs, Kinemation has a brand new LOD algorithm that supports LOD
Crossfade, including SpeedTree Crossfade. You can use the same LOD Group
GameObject Component when authoring. However, the runtime components are set up
differently. Most LOD users will be unaffected by this, other than seeing a
performance improvement.

Material Property components and shaders are all preserved. Even picking and
highlighting functions as normal.

Features and improvements made available with Kinemation include:

-   F and Shift + F for runtime entities (and closed subscenes) support
-   LOD Crossfade support
-   LOD Pack which allows packing up to 3 LOD levels (the last level can be an
    empty fade-out) into a single entity with crossfade for even better
    performance than LOD Group
-   Unique Meshes (easy runtime-generated meshes)
-   Exposed skeletons which automatically deform meshes based on bone entity
    transforms
-   Optimized skeletons which keep bones in buffers for fast manipulation
-   A runtime skinned mesh to skeleton binding system that doesn’t require
    copying a bindposes array
-   ACL animation compression and code-driven playback (both skeleton and
    arbitrary parameter tracks)
-   Bake-time support for animation retargeting
-   Animation clip event storage
-   Inertial blending
-   Root motion utilities
-   Automatic bounds updates
-   Squash-and-stretch bone scaling for Optimized Skeletons
-   Dynamic Meshes (animate vertices using Burst)
-   Flexible baking APIs
-   API for managing custom compute shader operations in the culling loop
-   Render visibility feedback
-   Jobified probes update optimization
-   Material property upload culling optimization
-   Highly optimized multi-mesh culling and LOD-aware deformation pipeline (both
    CPU and GPU benefits)
-   Vertex skinning support (faster on some platforms when simple skinning is
    needed)
-   Dual Quaternion Skinning
-   Custom shader graph nodes with motion history (history only works with
    optimized skeletons, and Unity’s built-in shader graph nodes are still
    supported)

### Calligraphics

Calligraphics makes no modifications to existing workflows.

Features made available include:

-   World-space text renderer with TextCore font support
-   Support for material property components and custom shader graphs
-   Animated properties and rich text tags
-   Changing text at runtime

### Unika Default Entity Remapping Systems

When entity references in Unika scripts are loaded from subscenes or when an
entity with scripts is instantiated, scripts need to explicitly serialize entity
references to a separate buffer so Unity can remap them, and then deserialize
the references afterwards. The default entity remapping systems will look for
enabled `UnikaEntitySerializationController` instances and serialize such
entities at the beginning of `InitializationSystemGroup` and deserialize them at
the end of `InitializationSystemGroup`.

## Changes When Enabling LifeFX

LifeFX is disabled by default with the installers commented out in the
bootstrap, because it introduces a partial sync point at the end of
`PresentationSystemGroup`. This is because VFX Graph updates between
`PresentationSystemGroup` and rendering and buffers need to be fully synced with
the main thread before that point.

## Changes When Enabling GameObjectEntity

Several additional systems will be added to support binding GameObjects with
entities at runtime and synchronizing their transforms.

## Changes When Using the NetCode Standard Injection Bootstrap

This bootstrap brings all the same changes as the Unity Transforms Injection
bootstrap. In addition, you need to replace all static method calls to
`ClientServerBootstrap` to instead use the `LatiosClientServerBootstrap`
equivalents.

Features made available include:

-   Custom bootstraps for each type of world that are automatically invoked when
    a world of that type is being created

## Changes When Switching to an Explicit Workflow Bootstrap

This changes how systems are injected and ordered in the world. Attribute
ordering is reserved for Unity systems and your top-level groups
(`RootSuperSystems`), which then explicitly specify the systems and groups they
update and the order they update them.

This is an opinionated feature that some people like and some don’t. A big
misconception is that you have to use it. You don’t. The framework works with or
without it. It is purely a preference. Who doesn’t like options?

## Changes When Enabling zeroToleranceForExceptions

This is a property on `LatiosWorld` you can set to `true` in the bootstrap for
each world. When enabled, if an exception occurs in a system and is caught by
the containing `ComponentSystemGroup`, all systems will stop executing to
prevent further error spam.

It can be a useful debugging feature.

## Changes When Enabling LinkedEntityGroup Length 1 Removal

Most bootstrap templates have this line commented out in the baking bootstrap.
But it is usually safe to enable. When enabled, prefabs which don’t have
children or other linked entities will have `LinkedEntityGroup` removed, unless
they were baked with `LinkedEntityGroupAuthoring`.

## Changes When Enabling Psyshock Bakers

This is another commented-out line of code in the baking bootstrap you can
uncomment to enable. This will cause Unity Engine colliders to be automatically
baked into Psyshock’s collider types. These can co-exist with Unity Physics
colliders.

Besides fattening your archetypes, there’s no real workflow differences with
this feature.

Features made available include:

-   Baking sphere colliders
-   Baking capsule colliders
-   Baking box colliders
-   Baking convex and concave mesh colliders
-   Baking compound colliders when multiple sphere, capsule, and/or box
    colliders belong to the same Game Object

Some people like to use Psyshock for trigger queries and Unity Physics for
simulation. And this feature can make authoring trigger shapes easier.

## Changes When Adding a Standard Bootstrap (QVVS Transforms)

Welcome to the deep-end! Maximum performance, ergonomics, and expression can be
found here. But not many venture this far.

The Standard bootstraps (both injection and explicit workflow variants) are
designed for using QVVS Transforms. As such, make sure to not use the
LATIOS_TRANSFORMS_UNITY scripting define symbol when using these.

With QVVS Transforms enabled, the transform system is completely swapped out.
This breaks compatibility with Unity Physics, parts of NetCode, and vanilla
Entities Graphics. You must use Kinemation for rendering.

This mode is best-suited for single-player experiences or if you are rolling
your own networking solution. It is especially well-suited for high entity
counts such as in top-down strategy games. Until others in the ecosystem start
adding optional support for QVVS Transforms, you are much more limited to what
the framework has to offer. However, you get improved synergy and performance of
features across the framework, as well as example projects and extra technical
support.

QVVS Transforms leverage the same `TransformUsageFlags` system during baking to
compute the correct components to apply. They also use a `Parent` component for
parenting, and a cached `Child` buffer. However, when writing transforms, you
always want to use `TransformAspect`, because it keeps local and world
transforms in-sync.

Early experimental NetCode support for QVVS Transforms was added in 0.11.0, and
slightly improved for 0.12.0. However, development will likely not continue due
to NetCode’s botched interpolation management.

Features and improvements made available include:

-   Stretch (shear-resistant non-uniform scaling)
-   GameObjectEntity (always on)
-   Motion History
-   Hierarchy Update Modes (lock world-space attributes on children)
-   World-space persistence when deparenting
-   `CopyParentWorldTransformTag`
-   Greatly improved chunk occupancy for root entities
-   Extreme Transforms mode for high entity count optimization
-   Parenting and reparenting system jobification optimizations
-   Improved determinism (for now)
-   Greatly improved performance accessing world-space rotation (affects all
    modules)
-   Improved ergonomics with other framework APIs
-   Automatic stretching of Psyshock colliders
-   GPU idle-cycle matrix conversion optimization
-   Mesh winding calculation optimization
-   GPU buffer packing optimizations for both renderables and bones
-   Reduced world-space transform memory bandwidth requirements
-   Squash-and-stretch bone support for exposed skeletons
-   Correct stretch of exported bones from optimized skeletons
-   Motion history support for exposed skeletons

## Changes When Enabling Scene Management

You can install this feature via `CoreBootstrap.InstallSceneManager()` in your
bootstrap.

If you just want simple synchronous scene loading and synchronous subscene
loading, this feature has you covered. It obviously is not suited for every
project, so only enable it if you think it is a good fit. It can be especially
nice for game jams.

The feature is prone to breaking third-party assets, which is why it is not
listed earlier.

Features made available include:

-   Automatic synchronous loading of a scene and all of its subscenes
-   Skips all execution of systems once the scene loads and starts over from the
    beginning of the frame where critical initialization systems can run with
    loaded entities
-   Automatically resets the `sceneBlackboardEntity`
-   Automatically invokes `ISystemNewScene` callbacks

Since 0.10, there are methods to allow specifying some subscenes to load
asynchronously. You can use these to force essentials to load first, but then
stream non-essentials in later.

## Frequently Asked Questions

*Q: What about singletons?*

If you want to use singletons, use singletons. They are never totally replaced.
Blackboard entities and collection components provide an alternative that you
may find to be more flexible. But you don’t have to use those features.

*Q: Is there a following?*

There is. Most of us have congregated to the framework discord, and don’t really
communicate much elsewhere.

*Q: What if I don’t want to use centralized system ordering?*

Since you skipped that part, that feature is completely optional and you can use
everything else without it. It is purely a preference.

*Q: How much of the framework works in Unity Transforms mode?*

Nearly all of it. What doesn’t work tends to be very specialized features. But
you will also miss out on some of the optimizations and ergonomics QVVS
Transforms offers. Even still, many people find the Latios Framework with Unity
Transforms worthwhile.
