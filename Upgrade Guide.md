# Upgrade Guide [0.13.7] [0.14.0]

If you are not using version control, please back up your project in a separate
directory before upgrading! You may need to reference the pre-upgraded project
during the upgrade process.

This document only highlights the common things to be aware of when upgrading.
See the changelogs of each module for a complete list of changes.

## Core

### Math Moved Out

All of the math-related APIs in Core, such as SIMD, RNG, QCP, and various other
utilities and extensions have been moved to the new Calci module. Simply
reference the new module in your assembly and add a `using Latios.Calci;` to
your files, and things should be back to normal.

## Psyshock

### Physics to TrueSim

Some static methods and constants which used to be in the Physics class are now
in the `TrueSim` class. They otherwise act the same as before.

## Myri

### AudioClipBlob Compression Support

Myri no longer exposes the raw samples in the `AudioClipBlob`, as they may now
be optionally stored in a compressed form. If you need APIs to extract the
audio, please make a feature request.

### SampleUtilities Renamed to DspTools

The static class `SampleUtilities` was renamed to `DspTools` to better reflect
the variety of features this class will provide.

## Kinemation

### Major Bounds Overhaul

Renderer bounds handling received a major overhaul. Deforming renderers now use
`WorldRenderBounds` and `ChunkWorldRenderBounds` like all other entities. In
addition, systems responsible for computing these bounds have been moved to run
after `LatiosEntitiesGraphicsSystem`, which is much later than when Entities
Graphics normally performs these tasks. Update any manually-created archetypes
and system orders accordingly.

### LODs Update

MMI Range LODs no longer contain a `height` field. They use the
`WorldRenderBounds` now. Just delete that field from custom bakers and runtime
code. Additionally, if you were adding `LodCrossfade` to entities yourself in
bakers, please use `MeshRendererBakeSettings.requireLodCrossfade` instead, as
otherwise you may get baking errors.

### Skeleton Culling Changes

Skeletons are no longer directly culled, and therefore skeletons without skinned
meshes will no longer have `RenderVisibilityFeedbackFlag` updated. A
`CullingGroup`-like API may be provided in the future to address this feature
regression, but the implementation of such a feature will be based on user
requests for it. If you are missing this functionality, please share your use
case through one of the various communication channels.

Additionally, Unity Transforms mode no longer requires the use of
`PostProcessMatrix` in any scenario.

## New Things to Try!

### Core

`UnsafeParallelBlockList` now has generically typed versions, making it easier
to express your intent when you don’t need type punning.

### New Module – Calci

Calci is a new module focused on math and algorithms. In addition to housing
former Core APIs, it comes with new Bezier curve APIs.

### Psyshock Physics

Psyshock now has a much more extensible API for drawing outlines of colliders,
allowing you to feed lines into your own line renderer.

### Kinemation

Kinemation now supports Mesh LODs for Unity 6.2 and newer. This means there are
now 3 different LOD systems which are able to crossfade. And now, these systems
coordinate with each other so that you can combine multiple techniques. You can
use Mesh LODs up until at a distance where you use LOD Pack to switch to an
imposter or fade out. And you can combine Mesh LOD with LOD Group for your
skinned meshes.

On the animation side, Kinemation’s new `UnityRig` class provides 2-bone IK. Use
this in combination with `OptimizedSkeletonAspect`’s new multi-bone writing APIs
to commit your IK changes in batch with ensured correctness.
