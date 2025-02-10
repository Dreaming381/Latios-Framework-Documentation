# LatiosWorld

Latios.Core provides quite a few unique features, many of which depend on
`LatiosWorld` being the primary `World`. But why? What is actually going on?
While you could look at the code to find answers, that code is a little scary…

Ok, probably really scary. It tends to fluctuate with each feature release as
things continue to improve.

Regardless, I will do my best to explain what `LatiosWorld` does and what
systems it automatically populates itself with and the roles they play.

## Default Systems

`LatiosWorld` automatically creates several top-level systems from within its
constructor:

-   InitializationSystemGroup
-   SimulationSystemGroup
-   PresentationSystemGroup (except in NetCode servers and thin clients)

For NetCode projects, it instead creates the appropriate Client and Server
variants. This is dictated by the `WorldRole` argument in the `LatiosWorld`
constructor.

`InitializationSystemGroup` has a special `IRateManager` attached to it which
modifies their behavior to support additional features.

## System Creation in LatiosWorld

The `InitializationSystemGroup` is the home to all Latios.Core systems.

The following systems are created by the `LatiosWorld`:

-   LatiosWorldUnmanagedSystem – This system manages the lifecycle of
    `LatiosWorldUnmanaged`. It has no per-frame update and does not belong to
    any group. It is the first system created.
-   [PreSyncPointGroup](Custom%20Command%20Buffers%20and%20SyncPointPlaybackSystem.md)
    – This is a `ComponentSystemGroup` designed for systems which schedule jobs
    asynchronous to the sync point.
-   [SyncPointPlaybackSystem](Custom%20Command%20Buffers%20and%20SyncPointPlaybackSystem.md)
    – This is a system capable of playing back command buffers which perform ECS
    structural changes.
-   SyncPointPlaybackSystemDispatch – This is a manager for
    `SyncPointPlaybackSystem` used for capturing exceptions during playback in
    the Editor.
-   [MergeBlackboardsSystem](Blackboard%20Entities.md) – This system merges
    `BlackboardEntityData` entities into the `sceneBlackboardEntity` and
    `worldBlackboardEntity` and is also responsible for creating the
    `sceneBlackboardEntity` and performing `OnNewScene()` callbacks
-   [ICollectionComponentsReactiveSystem and
    ManagedStructComponentsReactiveSystem](Collection%20and%20Managed%20Struct%20Components.md)
    – These systems synchronize `ICollectionComponent` and
    `IManagedStructComponent` types with the presence of the `ExistComponent` of
    each type.
-   LatiosWorldSyncGroup – This is a `ComponentSystemGroup` designed for
    application code. Its purpose is to provide a safe location in
    `InitializationSystemGroup` for application code to execute. Since many
    systems in `InitializationSystemGroup` generate structural change sync
    points, it is common to place systems that induce structural change sync
    points via batch `EntityManager` calls here and treat
    `InitializationSystemGroup` as a mega sync point. Reactive systems should
    use `OrderLast = true` to see all the changes made by previous systems.
-   PostSyncPointGroup – This is a `ComponentSystemGroup` that updates at the
    very end of `InitializationSystemGroup` and is a dedicated place to schedule
    cleanup jobs after all the structural changes have happened
-   [AutoDestroyExpirablesSystem](Auto-Destroy%20Expirables.md) – This system is
    responsible for detecting expired entities and queueing them up for
    destruction in `SyncPointPlaybackSystem`.
-   [SceneManagerSystem](Scene%20Management.md) (if installed) – This system
    changes scenes and updates the `CurrentScene`. It also initiates the pauses
    all remaining systems in the frame until the scene loads.
-   DestroyEntitiesOnSceneChangeSystem (if installed) – This system destroys
    procedural entities whenever the scene changes. It is updated via scene
    change events rather than a typical `ComponentSystemGroup`.

## System Ordering in InitializationSystemGroup

Currently, the `LatiosInitializationSystemGroup` orders itself as follows:

-   PreSyncPointGroup
-   AutoDestroyExpirablesSystem
-   SyncPointPlaybackSystemDispatch
    -   SyncPointPlaybackSystem
-   BeginInitializationEntityCommandBufferSystem
-   SceneManagerSystem (if installed)
-   [End OrderFirst region]
-   …
-   [Unity SceneSystemGroup]
-   LatiosWorldSyncGroup
    -   MergeBlackboardsSystem
    -   ManagedStructComponentReactiveSystem
    -   CollectionComponentReactiveSystem
    -   [End OrderFirst region]
-   …
-   EndInitializationEntityCommandBufferSystem
-   PostSyncPointGroup

## LatiosWorld Creation in Detail

When a `LatiosWorld` is created, it performs a sequence of setup steps.

First, it registers a custom `ComponentSystemGroup` update delegate which allow
all `ComponentSystemGroup`s to support Latios Framework features.

Second, it scans the list of unmanaged systems and generates generic classes for
`ISystemShouldUpdate` and `ISystemNewScene`.

Third, it creates a `LatiosWorldUnmanagedSystem`, which is an unmanaged system
that does not update but rather governs the lifecycle of the
`LatiosWorldUnmanagedImpl`. During the system’s `OnCreate()`, it creates for the
impl an instance of `CollectionComponentStorage`. As its name implies, this
object stores the collection components that can be attached to entities.
`CollectionComponentStorage` also stores the `JobHandle`s with each collection
component. `CollectionComponentStorage` can be initialized fully within Burst.
`ManagedStructComponentStorage` on the other hand is lazily initialized to
ensure it is initialized in a non-Burst-compiled context, and is not constructed
during the system `OnCreate()`.

Fourth, the `LatiosWorldUnmanagedSystem` creates `worldBlackboardEntity`.

Fifth, it creates a cache of the collection component dependencies pending
update of an executing `SystemState.Dependency`’s final value. The cache is
stored in the impl.

Finally, it creates the `InitializationSystemGroup`, the
`SimulationSystemGroup`, and the `PresentationSystemGroup`. The
`InitializationSystemGroup` is given a custom `IRateManager` which creates the
remaining essential systems and injects them into the
`InitializationSystemGroup`.

The `LatiosWorld` contains a couple of flags used for stopping and restarting
simulations of systems on scene changes. All `ComponentSystemGroups`
conditionally check one of these flags. This behavior is only used when the
Scene Manager is installed.

The `LatiosWorld` also contains the public `useExplicitSystemOrdering` flag
which tells `SuperSystem`s if they should enable system sorting by default. This
is used by the bootstrap templates to set the appropriate workflow. However, a
`SuperSystem` may override this setting for itself in `CreateSystems()` by
setting the `EnableSystemSorting` flag.

And lastly, The `LatiosWorld` contains a public `zeroToleranceForExceptions`
flag which will automatically stop all system execution when an exception is
caught by one of the Latios Framework system dispatchers. This can be quite
useful for debugging.
