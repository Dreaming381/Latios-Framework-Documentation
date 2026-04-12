# Upgrade Guide [0.14.15] [0.15.0]

If you are not using version control, please back up your project in a separate
directory before upgrading! You may need to reference the pre-upgraded project
during the upgrade process.

This document only highlights the common things to be aware of when upgrading.
See the changelogs of each module for a complete list of changes.

## Core

### IAspect Removal

Unity has deprecated `IAspect` for a while now, and it is removed in Unity 6.5.
Latios Framework 0.15.x is the first version to formally remove `IAspect`
support. Now, Core defines its own `IAspect` type which will be expanded over
time to bring back some of the functionality of Unity’s old `IAspect` type. Some
types that used to implement Unity’s `IAspect` still exist, but can no longer be
used in source generated entity iteration.

### Live Baking Changes

`SystemState.GetLiveBakeSafeLastSystemVersion()` has been removed, as it has
been replaced with a much more powerful set of live bake tracking APIs.

## Psyshock

### Physics to TrueSim

Some static methods and constants which used to be in the Physics class are now
in the `TrueSim` class. They otherwise act the same as before.

## Transforms

### worldIndex Renamed to context32

This field on `TransformQvvs` has been renamed to better reflect how the
variable is used in practice.

### QVVS V2

The QVVS Transform system runtime (ECS components and systems) have been
completely overhauled. There’s a dedicated upgrade guide
[here](Transforms/QVVS%20V2%20Upgrade%20Guide.md).

## Kinemation

### OptimizedSkeletonAspect Redesign

`OptimizedSkeletonAspect` has been redesigned. With the removal of Unity
`IAspect`, this type can now be constructed explicitly, which allows it to do
initialization automatically. Thus, `ForceInitialize()` is gone. Also, when
using QVVS Transforms, the type now requires a mutable `TransformAspect` and a
`SocketLookup`, because sockets are evaluated immediately. This also means that
it is possible to modify the skeleton’s transform in the same job the skeleton
is modified, so `OptimizedRootDeltaROAspect` has been removed.

`DependentSkinnedMesh` and `SkeletonDependent` are now public API, replacing
`SkinnedMeshBindingAspect` and `SkeletonSkinBindingsAspect` for custom graphics
purposes.

### New Culling Pipeline Changes

The system order has been reworked to better optimize for projects using LifeFX,
and to prepare for improved batching management in the future. The custom
graphics pass is now always enabled, meaning `EnableCustomGraphicsTag` has been
removed. However, deformations and other dispatch systems are disabled by
default. They can be enabled via `EnableUpdatingInCustomGraphics` on the
`worldBalckboardEntity`.

Additionally, UniqueMesh now uploads before culling, in accordance to Unity’s
updated `BatchRendererGroup` documentation. Some flags and bootstrap APIs have
been removed as a result.

## Calligraphics

### Calligraphics V2

Calligraphics has been heavily overhauled with the latest TextMeshDOTS changes.
There’s a dedicated upgrade guide
[here](Calligraphics/Calligraphics%20V2%20Upgrade%20Guide.md).

## New Things to Try!

### Core

You can add the `[DontSyncPreviousUpdatesThisFrame]` attribute on your systems
to avoid partial sync points for systems that update multiple times per frame.

There’s also a new feature VPtr which is an advanced API for polymorphic struct
pointers that supports implementations across multiple assemblies. It works
similarly to Unika interfaces.

### New Module – Aux ECS

Aux ECS is a complimentary single-threaded ECS for when you need to track
additional data on entities without invoking structural changes in Unity’s ECS.

### Calci

The Latios Framework finally has an official `NativePriorityQueue` type, which
lives in Calci. It is a quaternary comparison-based heap queue.

### Kinemation

Kinemation now supports mipmap texture streaming for all baked entities. This
works silently out-of-the-box. Just enable mipmap streaming on your textures,
and let Kinemation do the rest!
