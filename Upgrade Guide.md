# Upgrade Guide [0.10.7] [0.11.0]

Please back up your project in a separate directory before upgrading! You will
likely need to reference the pre-upgraded project during the upgrade process.

This document only highlights the common things to be aware of when upgrading.
See the changelogs of each module for a complete list of changes.

## Core

### New Unity Packages

All Unity ECS Packages have been updated to 1.3.2 versions. You will need to
upgrade these packages, which may introduce some behavioral differences.

## Kinemation

### LOD Group Baking Override API

If you used `MeshRendererBakeSettings` and specified a `lodSettings`, this field
no longer exists. The logic for this has been refactored to
`RenderingBakingTools.BakeLodMaskForEntity()`.

### New Culling and Dispatch

If you’ve done anything with the `GraphicsBufferBroker` or hooked into the
culling API, some things have changed in order to facilitate the various
performance improvements made with this release. Check the changelog to see the
list of breaking changes.

### MeshDeformDataBlob Use Case Data Stripping

Lastly, it is no longer safe to assume specific attributes of
`MeshDeformDataBlob` are populated. There are new has\* properties to help you
determine whether data is correctly populated. In common cases, much of the
blob’s data was never used, so feature flags are now used during baking to
determine what gets populated. This can also result in the same mesh having
multiple deriving `MeshDeformDataBlob` instances in incremental baking, though
this should never happen in a full subscene bake.

## New Things to Try!

### Compatibility

Some new experimental compatibility improvements have been added for NetCode,
including explicit system ordering workflow support and basic QVVS
`WorldTransform` replication.

### Core

QCP is a powerful algorithm for aligning objects based on point pairs. It is
similar to the Kabsch algorithm, but much faster.

### Psyshock Physics

All layer queries (`DistanceBetween()`, `Raycast()`, `ColliderCast()`, and
`FindObjects()`) now have overloads which take a `ReadOnlySpan<CollisionLayer>`
allowing multiple layers to be queried at once. The `layerIndex` corresponding
to the index in the span can be found in the result.

### Kinemation

Add a *LOD1 Append* authoring component to any *Mesh Renderer* to turn it into a
2-LOD group within a single entity at runtime. This saves all the headache of
juggling child transforms, updating child world bounds, and frustum culling each
LOD level. It can be an amazing performance boost for frequently-instantiated
projectile entities.

There’s some new root motion utility APIs in the form of
`RootMotionDeltaAccumulator` and `RootMotionTools` to help you sample and blend
root motion transform deltas.

Doing anything with custom graphics? Check out `GraphicsUnmanaged` and
`GraphicsBufferUnmanaged` to perform graphics operations in Burst-compiled code!
