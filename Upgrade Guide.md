# Upgrade Guide [0.9.4] [0.10.0]

Please back up your project in a separate directory before upgrading! You will
likely need to reference the pre-upgraded project during the upgrade process.

This document only highlights the common things to be aware of when upgrading.
See the changelogs of each module for a complete list of changes.

## Core

### New Unity Packages

All Unity ECS Packages have been updated to 1.2.1 versions. You will need to
upgrade these packages, which may introduce some behavioral differences.

You will need to add the `ENTITY_STORE_V1` scripting define to your project.

### UnsafeParallelBlockList

Auxiliary types such as `ElementPtr` and `Enumerator` now live in
`UnsafeIndexedBlockList` instead.

## Psyshock

### CollisionLayer Backing

CollisionLayer is now backed by NativeLists instead of NativeArrays, which can
result in some safety errors at runtime that used to be perfectly safe.

### FindPairs Changes

`IFindPairsProcessor.BeginBucket()` returns a `bool`, which should be `true` to
preserve previous behavior. Returning `false` lets further processing of the
bucket context to be skipped.

Additionally, `PhysicsComponentLookup` and similar types no longer accept the
`SafeEntity` from a `ScheduleParallelUnsafe()` context. Use the normal lookup
types with `[NativeDisableParallelForRestriction]` instead.

## Kinemation

### LOD Group Rewritten

If you worked with LOD Groups in code at all, you will have to rework that code
because Kinemation has a brand new high-performance LOD Group algorithm which
supports LOD Crossfade including SpeedTree Crossfade.

If you only use LOD Group in the Editor, you shouldn’t be impacted.

`MeshRendererBakeSettings` and associated custom baking APIs are affected by
this change. There’s a new `LodSettings` `struct` to replace `lodGroupEntity`
and `lodGroupMask`.

### New Bone Animation Weighting Scheme

In 0.9, it was expected that when blending bones, the total weights accumulated
to 1f for each bone. In 0.10, the accumulated weight is stored in the
`worldIndex` of the `TransformQvvs` and is automatically written or added to
when performing sampling. The accumulated weight can be any positive non-zero
value for normalization to function correctly. But you should be wary of this in
relevant code. A big motivation for this change was to simplify masked sampling
support for additive and override layers.

`BufferPoseBlender` APIs have changed as a result.

## Calligraphics

### New TextBaseConfiguration

Calligraphics received a major overhaul, and the base text renderer
configuration has many more options you will need to initialize. Additionally,
some advanced code usages, such as reading the `FontBlob` directly, will require
significant code changes.

## New Things to Try!

### Core

`AddComponentsCommandBuffer` is here. People often request more of these
high-performance command buffers. The motivation for implementing this one was
to handle an edge case with cleanup components more elegantly.

### QVVS Transforms

GameObjectEntity is now supported for Unity Transforms.

### Psyshock Physics

`UnitySim` now has all the APIs required to build a functional rigid body
simulation based on Unity Physics solver. You will also want to leverage the new
`PairStream`, `IForEachPairProcessor`, and `Physics.ForEachPair()` to build out
your constraint solver with safety, flexibility, and full parallelism.

### Kinemation

Kinemation now has dual quaternion skinning, new APIs for sampling masked
subsets of bones and parameters, and LOD Crossfade support.

### Calligraphics

Calligraphics received a major overhaul to its glyph generation engine,
supporting many more features of TextMeshPro including many more rich text tags,
base formatting options, and multi-font support.

Additionally, Calligraphics has a new GPU-Resident option which allows for
caching the generated glyphs on the GPU for text that changes infrequently.
