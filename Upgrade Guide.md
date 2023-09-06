# Upgrade Guide [0.7.7] [0.8.0]

Please back up your project in a separate directory before upgrading! You will
likely need to reference the pre-upgraded project during the upgrade process.

## Core

### New Bootstrap and Installers

You may wish to update your bootstrap to include the installers for new features
and modules.

### OnNewScene Callbacks

`OnNewScene` callbacks have been redesigned to occur at a fixed point in the
frame just after blackboard entities have been merged after a scene change. This
allows the callbacks to react to per-scene options. You may have to rework
initialization logic to account for this change.

## Psyshock Physics

### Two Mesh Blob Types

With the addition of TriMesh colliders, instead of calling
`RequestCreateBlobAsset()` in a baker, you must now call
`RequestCreateConvexBlobAsset()` or `RequestCreateTriMeshBlobAsset()`.

## Kinemation

### Mecanim Controllers

If you recreated a new bootstrap, you will need to verify that all Animator
components do not have an Animator Controller if you wish to preserve the old
behavior. You can also disable the Mecanim features in the bootstrap by removing
the associated lines.

### Root Motion Baking

Clips are always baked with root motion by default, which causes root bone
motion in the model to always be applied to the root bone of the skeleton (which
may be an ancestor). There is a setting to override this behavior.

### ExposedBoneInertialBlendState Removed

This component was removed, as some users may prefer to store this data
differently, such as in a dynamic buffer paired with the `BufferPoseBlender`
API. You can create a component of the same name and give it an
`InertialBlendingTransformState` field.

### OptimizedSkeletonAspect Return Changed

The property `skeletonWorldTransform` returns by value instead of by `ref
readonly`. This change was unfortunately necessary to facilitate Unity
Transforms compatibility.

## New Things to Try!

### Unity Transforms Support

By setting LATIOS_TRANSFORMS_UNITY in your scripting define symbols, you can
enable Unity Transforms compatibility mode. Fancy transform features are not
supported in this mode, so only use this if you only need standard position and
rotation transform operations that Unity provides.

### Core

Smart Blobbers have a new `ISmartPostProcessItem` which allows you to create a
dynamic number of items to be later resolved from inside a `Baker`.

### QVVS

`GameObjectEntity` lets you convert a `GameObject` in the scene to an entity at
runtime. It does not use baking, but it will create a `WorldTransform` to sync
with so that entities can reason about the `GameObject’s` position. There’s an
interface your `MonoBehaviour`s can implement to configure the Entity, including
setting up `IManagedStructComponent`s. Additionally, you can make a
`GameObjectEntity` target a `GameObjectEntityHost` inside a subscene, and the
entities will get merged at runtime, so you can combine baking and runtime
conversion.

Hierarchy Update Modes let you lock world-space properties on child entities
before updating the hierarchy, which may cause the local-space properties to get
updated again. You can mix and match properties at per-axis granularity.

### Psyshock Physics

TriMesh colliders are here, for all your concave mesh collider needs. Simply
bake a normal Mesh Collider with *Convex* unchecked.

You can now use `FindObjects` in a `foreach` statement, which makes for much
more ergonomic environment scanning.

### Kinemation

There’s now a Mecanim state machine baking and runtime built right into
Kinemation. This supports all blend tree types, interrupts, code-driven blends,
events, and root motion. You can access all of this at runtime using the
`MecanimAspect`.

### Calligraphics

New module!

This module contains a world-space text rendering suite with a built-in tweening
engine while using the standard ECS rendering workflow. It is great for things
like overhead character names or damage numbers, but there are plenty of things
you can do with it!
