# ISystem Support

The Latios Framework has great support for `ISystem` with utilities to provide
the same capabilities as SubSystem, but for Burst-compatible systems. This page
lists out the various features which are expected to work in `ISystem` and how
to utilize them.

## OnNewScene and ShouldUpdateSystem

These exist via the interfaces `ISystemNewScene` and `ISystemShouldUpdate`. A
struct implementing these should also implement `ISystem`. Currently these
functions do not support Burst compilation.

## Fluent Queries

Fluent query API can be accessed via the `SystemState.Fluent()` extension
method.

## Latios World

You can get a subset of the `LatiosWorld` API by calling
`SystemState.GetLatiosWorldUnmanaged()`. It is recommended you do this during
`OnCreate()` and cache the returned instance in a member field for later use.
Acquiring the `LatiosWorld` can be Burst-compiled.

## Blackboard Entities

Blackboard entities can be acquired from a `LatiosWorldUnmanaged` instance.
`BlackboardEntity` methods have the same Burst support as the `EntityManager`
methods they mirror.

## Custom Command Buffers

Custom command buffers can be created, written to, and played back all within a
single Burst method. They can also be passed into or out of Burst-compiled
static methods just like any other native container.

## Collection Components

Collection components can be accessed from `LatiosWorldUnmanaged`. If one of the
associated methods is called from within a system updated by a `SuperSystem`
method (which is true for root-level `ComponentSystemGroup`s as well as all
Latios Framework-defined `ComponentSystemGroup` types), automatic dependency
management is performed using the `Dependency` property of `SystemState`. All
collection component operations are fully Burst-compatible.

The `ExistComponent` is supported by `SystemAPI` methods only if the
`ICollectionComponent` is defined in a separate assembly from the system.

## Managed Struct Components

Managed struct components can also be accessed from `LatiosWorldUnmanaged`.
However, these are not Burst-compatible. Still, they can be useful if only plan
to Burst-compile parts of an `ISystem`.

The `ExistComponent` is supported by `SystemAPI` methods only if the
`IManagedStructComponent` is defined in a separate assembly from the system.

## Sync Point

The Sync Point can be accessed via `LatiosWorldUnmanaged.syncPoint`. The same
automatic dependency management rules apply as Collection Components.

## Dependency Management

There is no difference to how automatic dependency management is handled between
managed and unmanaged systems. To manually update a system, use
`SuperSystem.UpdateSystem()`, which will ensure the correct dependency tracking
is performed.

## RNG

The `Rng` type works very well in Burst systems. The only pitfall is that it
can’t be seeded with a `string` in Burst. Additionally, you may find `SystemRng`
to be more useful when paired with `IJobEntityChunkBeginEnd`.
