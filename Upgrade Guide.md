# Upgrade Guide [0.11.5] [0.12.0]

If you are not using version control, please back up your project in a separate
directory before upgrading! You may need to reference the pre-upgraded project
during the upgrade process.

This document only highlights the common things to be aware of when upgrading.
See the changelogs of each module for a complete list of changes.

## Compatibility and Installation

### Minimum Versions Updated

The minimum version of Unity is now 6000.0.23f1. Support for 2022 LTS has been
dropped.

The minimum version of Entities is 1.3.9, the minimum version of Entities
Graphics is 1.4.6, and the minimum version of Burst is 1.8.19.

### New Bootstrap

The `ICustomEditorBootstrap` API has changed. Instead of passing an existing
world to replace, the interface now passes in a name string for a new world.
There are new bootstrap templates with these changes, as well as new module
installers.

## Core

### New Unity Packages

All Unity ECS Packages have been updated to 1.3.2 versions. You will need to
upgrade these packages, which may introduce some behavioral differences.

## Transforms

### Baking Typo Fix

`AddHiearchyModeFlags()` has been renamed to `AddHierarchyModeFlags()`. Your
inner pirate can rejoice that the treasured letter has been found.

## Kinemation

### Exported Bone is now Socket

The entity that would automatically track a bone within an optimized skeleton is
now called a “Socket” to reflect industry terminology and terminology Unity
started using at Unite 2024.

### Baking Interfaces Replace Base Classes

`OverrideMeshRendererBase` has been replaced with `IOverrideMeshRenderer` and
`OverrideHierarchyRendererBase` has been replaced with
`IOverrideHierarchyRenderer`. Your base class can just be a `MonoBehaviour` now.

### Culling Round-Robin is Now Dispatch Round-Robin

`CullingRoundRobinEarlyExtensionsSuperSystem` is now
`DispatchRoundRobinEarlyExtensionsSuperSystem`. Similarly,
`CullingRoundRobinLateExtensionsSuperSystem` is now
`DispatchRoundRobinLateExtensionsSuperSystem`. With Unity 6, dispatches are
batched and no longer directly related to culling passes.

## Mimic

Mimic has been removed as a framework module. Its contents can now be found in
the [Add-Ons package](https://github.com/Dreaming381/Latios-Framework-Add-Ons).

## New Things to Try!

### Core

`ComponentBroker` and `TempQuery` provide job-friendly equivalents for when you
need something similar to an `EntityManager` or `EntityQuery`. These are
especially useful for jobs that use scripting systems or complex call stacks.

Core contains some baking improvements. Extension methods allow bakers to get
components by interface with proper dependency registration. And
`WorldUpdateAllocator` is now safe to use in baking systems.

### QVVS Transforms

A new shader library provides a set of hlsl includes for working with QVVS
Transforms.

### Psyshock Physics

The `QuickTests` static class provides stateless query operations for special
use cases, such as working with ellipses or doing manual frustum culling.

### Myri

Myri has a new editor managed audio driver. This can be enabled per project per
device in the edit menu. When enabled, this driver circumvents a bug many
suffered where Unity would get stuck indefinitely during domain reloads after
code changes.

### Kinemation

At authoring time, you can now make a socket out of any entity. You no longer
have to use the rig importer settings for your character. Instead, just use the
Socket authoring component on a child of your Animator.

If you need an authoring time view of your optimized skeleton, perhaps to
process in a baker, you can now use an Optimized Skeleton Cache authoring
component. This makes it easier to bake IK target bone indices or similar
mechanics.

Need some bone masks? There is now a `SkeletonBoneMaskSetBlob` and associated
smart blobber. Notably, the smart blobber accepts `AvatarMask` as input, so you
can use Unity’s editor workflows to create your bone masks.

What used to be LOD1 Append is now called LOD Pack, and it now supports up to 3
levels. You can also leave the last level absent for a fade-out effect.

Making unique meshes at runtime that can be rendered as entities can be quite
complex. Ignore all that complexity and write all your mesh data into dynamic
buffers with Unique Mesh!

EWBIK is an inverse-kinematics algorithm that supports multiple position and
orientation targets simultaneously. The algorithm is implemented modularly so
that you can plug in your own constraint solvers.

### New Modules – LifeFX and Unika

This release contains two new modules for you to try!

LifeFX provides a mechanism for sending data from ECS into Game Object land via
graphics buffers. The most common use-case for this is to drive VFX Graph, and
LifeFX includes integration `MonoBehaviours` to do all the Game Object logic for
you. The system is designed to allow a single graph instance drive thousands of
entity effects similar to what is seen in Unity’s Galaxy Sample.

Unika provides a C\# scripting solution where scripts are packed into a dynamic
buffer and can be interacted with abstractly and polymorphically with the help
of source generators. It includes comprehensive serialization support for
scripts to reference other scripts, scripts to reference entities, and for ECS
components to reference scripts. And all of this is compatible with Jobs and
Burst!
