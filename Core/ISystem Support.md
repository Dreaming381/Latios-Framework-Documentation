# ISystem Support

The Latios Framework has great support for `ISystem` with utilities to provide
the same capabilities as `SubSystem`, but for Burst-compatible systems. This
page lists out the various features which are expected to work in `ISystem` and
how to utilize them.

## OnNewScene and ShouldUpdateSystem

These exist via the interfaces `ISystemNewScene` and `ISystemShouldUpdate`. A
struct implementing these should also implement `ISystem`. Currently these
functions do not support Burst compilation. They require the system belong to a
`LatiosWorld`.

When using the scene management system, a `ISystemNewScene` can receive a
callback when a new scene is created by overriding `OnNewScene()`. The order
these functions are called do not reflect the system order. This callback
happens after the subscenes have been loaded and the blackboard entities have
been merged.

`ShouldUpdateSystem()` is called on an `ISystemShouldUpdate` right before it
updates to dictate whether the `ISystem` should execute `OnUpdate()`. This does
not disable Unity’s `EntityQuery` checks.

*Caution: Dependency and version numbers have not been updated yet when*
`ShouldUpdateSystem()` *is invoked. Only* `EntityManager`*,* `EntityQuery`*,
and* `BlackboardEntity` *operations are recommended.*

## Fluent Queries

A Fluent query can be started by calling `state.Fluent()` on the passed in
`SystemState`. Fluent queries work in any ECS world.

## Latios World

You can get a Burst-compatible subset of the `LatiosWorld` API by calling
`state.GetLatiosWorldUnmanaged()`. It is recommended you do this during
`OnCreate()` and cache the returned instance in a member field for later use.
Acquiring the `LatiosWorldUnmanaged` can also be Burst-compiled.

If you use `ILatiosApi`, you can access the `LatiosWorldUnmanaged` through the
fetched `api` instead.

## Blackboard Entities

Blackboard entities can be acquired from a `LatiosWorldUnmanaged` instance or
directly from the `api`. `BlackboardEntity` methods have the same Burst support
as the `EntityManager` methods they mirror.

## Collection Components

Collection components can be accessed from `LatiosWorldUnmanaged`. If one of the
associated methods is called from within a system updated by a
`ComponentSystemGroup`, automatic dependency management is performed using the
`Dependency` property of `SystemState`. All collection component operations are
fully Burst-compatible.

Automatic dependency management means when fetching a [collection
component](Collection%20and%20Managed%20Struct%20Components.md), rather than
complete the `JobHandle`s associated with that component, the `JobHandle`s will
be combined with `state.Dependency`. Additionally, by default `state.Dependency`
will automatically be used to update the `JobHandle`s of the collection
component after the `ISystem` finishes `OnUpdate()`. When combined with
`IJobEach` or `IJobEntity`, you may never have to explicitly touch `Dependency`
nor any other `JobHandle`.

The `ExistComponent` is supported by `SystemAPI` methods only if the
`ICollectionComponent` is defined in a separate assembly from the system.

## Managed Struct Components

Managed struct components can also be accessed from `LatiosWorldUnmanaged`.
However, these are not Burst-compatible. Still, they can be useful if only plan
to Burst-compile parts of an `ISystem`.

The `ExistComponent` is supported by `SystemAPI` methods only if the
`IManagedStructComponent` is defined in a separate assembly from the system.

## Custom Command Buffers

Custom command buffers can be created, written to, and played back all within a
single Burst method. They can also be passed into or out of Burst-compiled
static methods just like any other native container.

## Sync Point

The Sync Point can be accessed via `api.syncPoint` or
`LatiosWorldUnmanaged.syncPoint`. As long as the `ISystem` is updated by a
`ComponentSystemGroup`, its `state.Dependency` after `OnUpdate()` will be
forwarded to `SyncPointPlaybackSystem`.

## Dependency Management

There is no difference to how automatic dependency management is handled between
managed and unmanaged systems. To manually update a system instead of from a
`ComponentSystemGroup`, use `SuperSystem.UpdateSystem()`, which will ensure the
same correct dependency tracking is performed.

## Disabling Multi-Update Sync Points

Unity ECS systems are designed to always sync the jobs they scheduled the
previous update. This prevents multiple frames of jobs being queued up and never
completing. However, for systems that may update multiple times per frame, such
as those which update in `FixedRateSimulationSystemGroup`, these sync points can
hurt performance.

In a `LatiosWorld`, you can add the `[DontSyncPreviousUpdatesThisFrame(int
maxUpdatesWithoutSync)]` attribute to any system. This will cause the system to
skip syncing after the first time it updates in a frame, up to the maximum
number of updates.

**Warning:** *The Dependency property is not guaranteed to account for all jobs
scheduled in the previous update. It only examines component read-write
accesses, and treats the previous update’s Dependency value as coming from a
different system. This can cause safety errors if your system only reads
components, but writes to a persistent collection stored on the system via a
job.*
