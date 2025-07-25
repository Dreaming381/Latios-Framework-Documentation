# Getting Started with Latios.Core Framework

## Runtime Settings

In *Project Settings*, under the *Player* tab, in *Other Settings* and under the
*Configuration* header, set the *API Compatibility Level* to *.NET Standard*.

Also, add the Scripting Define Symbols:

-   UNITY_BURST_EXPERIMENTAL_ATOMIC_INTRINSICS
-   ENTITY_STORE_V1

### Mono

No additional settings required.

### Il2Cpp

In the *Build Settings*, you may need to set the *IL2CPP Code Generation* to
*Faster (smaller) builds*

## Bootstrap

Latios Framework requires a custom bootstrap to inject its custom features into
the ECS runtime.

You can create one from one of the templates using the Project folder’s create
menu (or the top menu’s “Assets” tab) and selecting *Latios-\>Bootstrap*.

For beginners, it is recommended to choose the *Standard-Injection Workflow*
variant, as this matches the default Unity behavior while bringing in all Latios
Framework features.

For those who prefer explicit ordering of systems, you may find the *Explicit
Workflow* more enticing. This workflow will automatically inject Unity systems,
but then allow you to inject only top-level systems which set up children
systems in a top-down manner. This is often the preferred workflow for
experienced users. See [Super Systems](Super%20Systems.md) for more info on this
workflow.

There are also Unity Transforms variants of these bootstraps for projects that
use Unity Transforms. You will need to add LATIOS_TRANSFORMS_UNITY scripting
define to your project if you use one of these bootstraps.

**Warning:** If you have the Unity Physics, Character Controller, or Vehicles
package installed, those packages are only compatible with Unity Transforms mode
of the framework, and you must use a Unity Transforms bootstrap. QVVS
alternatives for some of these packages may be found in the Latios Framework
Add-Ons package at various stages of development.

For NetCode Projects, the Latios Framework will detect the NetCode package and
enable additional C\# files for compilation. One of these provides a dedicated
NetCode bootstrap type called `LatiosClientServerBootstrap`, which offers Latios
Framework equivalents of `ClientServerBootstrap`. You can further customize each
type of world with the additional bootstrap interfaces provided, which you can
automatically generate from the *NetCode Standard – Injection Workflow*
bootstrap. The *NetCode QVVS – Explicit Workflow* bootstrap is experimental.

After the bootstrap is created, it can be
[customized](Customizing%20the%20Bootstraps.md).

## Common Types

-   [LatiosWorld](LatiosWorld%20in%20Detail.md) – a `World` subclass which
    contains the extra data structures and systems required for the framework’s
    features to work
-   [LatiosWorldUnmanaged](ISystem%20Support.md) – a struct which provides a
    subset of the `LatiosWorld` API for Burst-compiled unmanaged systems
-   [SubSystem](Sub-Systems.md) – a `SystemBase` subclass with Latios Framework
    API conveniences, though most systems should be written with
    [ISystem](ISystem%20Support.md)
-   [SuperSystem](Super%20Systems.md) – a `ComponentSystemGroup` subclass that
    supports Latios Framework features
-   [RootSuperSystem](Super%20Systems.md) – a `SuperSystem` subclass designed to
    be auto-injected even when using explicit system ordering
-   [ICollectionComponent](Collection%20and%20Managed%20Struct%20Components.md)
    – a struct type component that can store NativeCollection types with an
    entity and tracks job dependencies automatically
-   [BlackboardEntity](Blackboard%20Entities.md) – an Entity with extensions to
    apply `EntityManager` operations on it
-   [Rng](Rng%20and%20RngToolkit.md) – a struct with a powerful thread-safe,
    deterministic, fire-and-forget workflow for random number generation
-   [Smart Blobber](Smart%20Blobbers.md) – a specialized baking API used for
    generating blob assets
-   [Component Broker](Component%20Broker.md) – a specialized type for accessing
    lots of component types generically in jobs
-   [Temp Query](Temp%20Queries.md) – a type that represents a temporary entity
    query created on the spot in a job

### Components

-   [WorldBlackboardTag](Blackboard%20Entities.md) – A tag component attached
    exclusively to the `worldBlackboardEntity`. You may choose to exclude this
    tag in an `EntityQuery`. Do not add or remove this component!
-   [SceneBlackboardTag](Blackboard%20Entities.md) – Same as
    `WorldBlackboardTag` but for the `sceneBlackboardEntity`. Do not add or
    remove this component!
-   [BlackboardEntityData](Blackboard%20Entities.md) – A configuration component
    which tells the Latios Framework to merge the entity this component is
    attached to into one of the two global entities. You are free to add and
    modify these components in your code.
-   [DontDestroyOnSceneChangeTag](Scene%20Management.md) – A tag component which
    preserves the Entity when the scene changes (actual scene, not just a
    subscene). You may add and remove these at will.
-   [RequestLoadScene](Scene%20Management.md) – Request a scene (true scene, not
    a subscene) to be loaded. Add or replace these to your heart’s content.
-   [CurrentScene](Scene%20Management.md) – This component is attached to the
    `worldBlackboardEntity` and contains the current scene, previous scene, and
    whether the scene just changed. Do not add, set, or remove this component!
-   [IAutoDestroyExpirable](Auto-Destroy%20Expirables.md) – This is an interface
    for components and buffers that allow for negotiated destruction of entities
    by independent systems

## Conventions

### Only one unique instance of a System

To understand why it is a bad idea to have multiple instances of the same type
of system (other than a `SuperSystem`), imagine you have a `PositionClampSystem`
that clamps the Translation component to a valid range. For performance reasons,
you specified a `ChangeFilter` on the `Translation` component so that you only
clamp values you did not clamp last frame.

Now what happens if you have two instances of `PositionClampSystem`? Well as
soon as the first instance sees a modified `Translation`, it writes to that
`Translation`. Then, the second instance sees that the `Translation` was
modified and also writes to it the clamped value. As the next frame rolls
around, the first `PositionClampSystem` sees that the very same `Translation`
was modified since the last time it ran. It was actually the second instance
that modified it, but the first instance doesn’t know that, so it modifies it
yet again. These two systems will ping-pong back and forth indefinitely, making
the `ChangeFilter` useless.

The correct solution to this problem is to instead have the same
`PositionClampSystem` update twice in the loop. That way, when the instance runs
the second time, it only compares changes to the first time it ran in the frame
rather than the previous frame. Likewise, the first time it runs in the frame,
it only compares changes to the second time it ran in the previous frame.
Ultimately, the behavior is as expected.

If using the *Bootstrap – Explicit Workflow*, you’ll find that this works
correctly out-of-the-box.

### Only one scene active at a time (When Scene Management is installed)

I know, I know. You’ve always had multiple scenes loaded at runtime because of
conflicts and editor performance and all that. But with subscenes, there’s no
need for that anymore. You can have as many subscenes as you like, but you
should keep scenes separate so that when you swap scenes, the slate can be wiped
clean.

### Keep Systems in Separate Assemblies from Components

This isn’t a hard-and-fast rule, but some of the Latios Framework custom
component types rely on source generators, and this can confuse `SystemAPI` if
they are in the same assembly as the systems that use them.
