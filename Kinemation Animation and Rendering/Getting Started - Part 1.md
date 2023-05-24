# Getting Started with Kinemation – Part 1

This is the third preview version released publicly. As such, it only provides
foundational features. For simple use cases, it is easy to get started. But for
advanced production workflows, that burden may fall on the user.

It will be up to you, a member of the community, to help guide Kinemation’s
continual development to meet the needs of real-world productions. Please be
open and loud on your journey. Feedback is critical.

This Getting Started series is broken up into multiple parts. This first part
covers the terms and runtime structure. It is okay if you do not understand
everything in this part or choose to skim through it. The remaining parts will
guide you through the process of configuring and animating a character as an
entity.

## Minimum Requirements

-   Currently Supported Platforms (out-of-the-box)
    -   Windows
    -   Mac OS
    -   Linux
-   GPU Minimum Requirements
    -   Supports Entities Graphics
    -   Supports 8 simultaneous compute buffers in compute shaders
    -   Has 32 kB of shared memory per thread group
    -   Supports thread group dimensions of (512, 1, 1)
-   Content Requirements
    -   Skeleton size must be 32767 bones or less
    -   All shaders require usage of special ECS skinning functionality
    -   No limit to number of meshes bound to skeleton, nor number of materials
        per mesh

## Skeletons and Bones – Not What You Think

Before we explore the different building blocks of Kinemation, we need to make
sure we are on the same page with these two terms. Kinemation has very peculiar
interpretations of the words “skeleton” and “bone”. These interpretations come
from Unity’s asset model. We’ll discuss this from the authoring perspective,
that is in terms of Game Objects.

A “bone” is simply a `Transform` that is included in a group of transforms for
animation purposes. Anything that originated from a `GameObject` can be a bone.
A foot can be a bone. A mesh can be a bone. A particle effect can be a bone.
Even the camera can be a bone. Some bones may influence skinned meshes. Some
won’t.

A “skeleton” is just a collection of bones. Those bones could be hierarchically
structured. But they also might not be. When evaluating skinning, bones are
transformed into the skeleton object’s local space. Skinned meshes bound to a
skeleton are forced to exist in this space. Because the bones and the meshes are
in the same coordinate space, skinning works.

The other aspect of a “skeleton” is that it is the bind target of skinned
meshes. Even if the skeleton contains multiple armatures from a DCC application,
the meshes get bound to the skeleton, not the armature `GameObjects`.

By default, Kinemation treats anything with an `Animator` as a skeleton. That
`GameObject` and all descendants become bones (except Skinned Meshes and child
Animator hierarchies). This means the “skeleton” is usually also a “bone”. For
optimized hierarchies, Kinemation creates a de-optimized shadow hierarchy and
analyzes that instead.

However, Kinemation also allows skeletons to be built procedurally from code.

## Anatomy of a Character at Runtime

There are ten main types of entities in Kinemation. Often, an entity may fall
under more than one type at once. The types are as follows:

-   Base skeleton
-   Exposed skeleton
-   Exposed bone
-   Optimized skeleton
-   Exported bone
-   Deformed mesh
-   Skinned mesh
-   Blend shape mesh
-   Dynamic mesh
-   Shared Submesh

Let’s break down each type.

### Base Skeleton

The *base skeleton* is an entity that can be a bind target for skinned meshes.
It provides a uniform interface for skinned meshes. It is composed of these
components:

-   SkeletonRootTag
    -   Required
    -   Zero-sized
    -   Added during baking
-   ChunkPerCameraSkeletonCullingMask
    -   Added at runtime automatically
    -   Chunk component used for culling
-   ChunkPerCameraSkeletonCullingSplitsMask
    -   Added at runtime automatically
    -   Chunk component used for culling shadow maps
-   SkeletonBindingPathsBlobReference
    -   Optional
    -   Added during baking
    -   Used for binding skinned meshes which don’t use overrides
-   DynamicBuffer\<DependentSkinnedMesh\>
    -   Internal
    -   Added at runtime automatically
    -   Cleanup type
-   SkeletonBoundsOffsetFromMeshes
    -   Internal
    -   Added at runtime automatically

A skeleton is not valid if it is only a base skeleton. It needs to also be
either an *exposed skeleton* or an *optimized skeleton*. Unless you are building
a skeleton procedurally, you generally do not need to concern yourself with any
of these components. However, you may find `SkeletonBindingPathsBlobReference`
useful for looking up bones by name.

A skeleton will automatically receive all motion history components during
baking.

### Exposed Skeleton

An *exposed skeleton* is a skeleton entity in which every one of its bones is an
entity, and bones rely on Unity’s transform system for skinning and animation.
In addition to the *base skeleton* components, it also has these components:

-   DynamicBuffer\<BoneReference\>
    -   Required
    -   Each element is a reference to an exposed bone entity
    -   Added during baking
-   BoneReferenceIsDirtyFlag
    -   Optional
    -   Used to command Kinemation to resync the exposed bones with the *exposed
        skeleton*
    -   Must be manually added if needed
    -   Zero-sized `IEnableableComponent`
-   ExposedSkeletonCullingIndex
    -   Internal
    -   Added at runtime automatically
    -   Cleanup type

In nearly all cases, the only thing you will want to do is read the
`BoneReference` buffer. The index of each bone in this buffer corresponds to the
bone index in a `SkeletonClip`.

For procedural skeletons, the `BoneReference` buffer is the source of truth. Its
state must be synchronized to all *exposed bones*. That happens once the first
time Kinemation sees the skeleton. It can happen again by adding the
`BoneReferenceIsDirtyFlag` component and enabling it. Keeping the
`BoneReferenceIsDirtyFlag` component around will cause a fairly heavy Kinemation
system to run every frame. Avoid using it unless you need it.

If an entity in a `BoneReference` does not have the `LocalToWorld` component,
bad things will happen.

The *exposed skeleton* entity is typically also an *exposed bone* entity.

### Exposed Bone

**Note: What Unity’s Rig tab in the model importer refers to as “exposed bones”
are consequently referred to as “exported bones” in Kinemation. This name better
suits the behavior of such bones as their transform data SHOULD NOT be modified
by user code. “Exposed bones” in Kinemation refer to bones whose transforms CAN
be modified by user code, and such modifications will be reflected in attached
Skinned Meshes. When discussing Kinemation issues, try to stick to the
Kinemation nomenclature.**

An *exposed bone* is an entity whose transform components dictate its state of
animation. It has the following components:

-   WorldTransform
    -   Required
    -   Added during baking
-   PreviousTransform
    -   Required
    -   Added during baking
-   TwoAgoTransform
    -   Required
    -   Added during baking
-   ExposedBoneInertialBlendState
    -   Optional
    -   Convenience for inertial blending operations
    -   Added during baking
-   BoneOwningSkeletonReference
    -   Optional
    -   References the *exposed skeleton* entity
    -   Added during baking (and initialized)
    -   Modified by Kinemation during sync if present
-   BoneIndex
    -   Optional
    -   The bone’s index in the `BoneReference` buffer
    -   Added during baking (and initialized)
    -   Modified by Kinemation during sync if present

Similar to `BoneReference`, you usually only want to read
`BoneOwningSkeletonReference` and `BoneIndex`. The latter is especially useful
when you need to sample a `SkeletonClip` while chunk-iterating bones.

In addition, an exposed bone may optionally have the following internal
components.

-   BoneCullingIndex
-   BoneBounds
-   BoneWorldBounds
-   ChunkBoneWorldBounds

These components are added automatically during baking but can also be added
using `CullingUtilities.GetBoneCullingComponentTypes()`. Every bone that
influences a skinned mesh **must** have these components, unless the culling
behavior is overridden entirely.

### Optimized Skeleton

An *optimized skeleton* is an entity whose bones are not represented as entities
for the purposes of animation and skinning. Instead, the bone data is stored
directly in dynamic buffers attached to the optimized skeleton. This keeps the
transforms of the bones next to each other in memory, making hierarchical
updates, frustum culling, and full pose animation significantly faster.

In addition to the *base skeleton* components, an *optimized skeleton* has the
following components:

-   DynamicBuffer\<OptimizedBoneTransform\>
    -   Required
    -   3 sets of local and root space QVVS transforms in rotation
    -   Added during baking (and initialized with the pose at baking time)
-   OptimizedSkeletonState
    -   Required
    -   Contains rotation status and various dirty flags
    -   Added during baking
-   OptimizedSkeletonHierarchyBlobReference
    -   Optional (Required for `OptimizedSkeletonAspect`)
    -   Provides hierarchical information to convert local space sampled
        animation into the skeleton’s local space
    -   Added during baking
-   DynamicBuffer\<OptimizedBoneInertialBlendState\>
    -   Optional (Required for `OptimizedSkeletonAspect`)
    -   Stores state required to perform inertial blending across the entire
        skeleton
    -   Added during baking

When interacting with *optimized skeletons*, it is highly recommended to use the
`OptimizedSkeletonAspect` as this provides an intuitive API for working with
animations and individual bones.

In addition, Kinemation automatically adds the following internal components at
runtime:

-   OptimizedSkeletonTag – Cleanup type
-   DynamicBuffer\<OptimizedBoneBounds\>
-   OptimizedSkeletonWorldBounds
-   ChunkOptimizedSkeletonWorldBounds

### Exported Bone

An *exported bone* is an entity that copies the root-space
`OptimizedBoneTransform` QVVS of an *optimized skeleton* bone and assigns it to
its own `LocalTransform` and `WorldTransform` (the latter for stretch). When
parented directly to the *optimized skeleton* entity, it effectively mimics the
transform of the optimized bone along with all the optimized bone’s animations.
This is often used for rigid character accessories like weapons or hats.

An *exported bone* has the following components:

-   WorldTransform
    -   Required
    -   Added during baking
-   LocalTransform
    -   Required
    -   Added during baking
-   BoneOwningSkeletonReference
    -   Required
    -   Specifies the skeleton entity to copy transforms from
    -   Added during baking
-   CopyLocalToParentFromBone
    -   Required
    -   Specifies the bone index to copy
    -   Added during baking

Kinemation will generate an exported bone for immediate children of the Animator
at baking time. Children which have a `SkinnedMeshRenderer` or an `Animator` or
do not map to a bone in the shadow hierarchy will not be baked into an *exported
bone*. You can also freely create *exported bones* at runtime by meeting the
above component requirements and specifying the bone to track in the
`CopyLocalToParentFromBone` component. The *optimized skeleton* does not know
about nor care about *exported bones*, which means you can have multiple
*exported bones* track the same optimized bone. Be careful though, because
*exported bones* can be relatively costly to update.

*Exported bones* update during the `TransformSuperSystem` update.

An *optimized skeleton* is **not** an *exported bone*.

### Deformed mesh

A *deformed mesh* is an entity whose mesh is deformed by skeletal animation,
blend shapes, or Burst jobs via Dynamic Mesh API. A deformed mesh must be at
least one of these three other types of meshes, but can also be more than one
type at once.

A *deformed mesh* has the following components:

-   WorldTransform
    -   Required
    -   Added during baking
-   LocalTransform
    -   Required
    -   Added during baking
-   MeshDeformDataBlobReference
    -   Required
    -   Contains all information Kinemation needs to deform the mesh on the GPU
    -   Added during baking
-   NeedsBindingFlag
    -   Optional
    -   Used to command Kinemation to resync mesh with deformers
    -   Must be manually added if needed
    -   Zero-sized `IEnableableComponent`
-   ShaderEffectRadialBounds
    -   Optional
    -   Adds additional bounds inflation for a mesh that modifies vertex
        positions in the vertex shader
    -   Must be manually added if needed
-   BoundMesh
    -   Internal
    -   Added at runtime automatically
    -   Cleanup type
-   ChunkDeformPrefixSums
    -   Internal
    -   Added at runtime automatically
-   Material Property Types
    -   Added during baking
    -   Each is optional and depends on shaders used
    -   Public
        -   CurrentMatrixVertexSkinningShaderIndex
        -   PreviousMatrixVertexSkinningShaderIndex
        -   TwoAgoMatrixVertexSkinningShaderIndex
        -   CurrentDeformShaderIndex
        -   PreviousDeformShaderIndex
        -   TwoAgoDeformShaderIndex
    -   Internal
        -   LegacyLinearBlendSkinningShaderIndex
        -   LegacyComputeDeformShaderIndex
        -   LegacyDotsDeformParamsShaderIndex

A *deformed mesh* does not necessarily need to be a renderable mesh, but if it
is and if the `MaterialMeshInfo` changes, then the deformed mesh must keep its
`MeshDeformDataBlobReference` up-to-date and rebind with `NeedsBindingFlag` on
every change.

The public material properties can be copied during culling passes for shaders
that use custom indices for deformations.

### Skinned mesh

A skinned mesh is a *deformed mesh* entity that can be deformed by a skeleton.
Skinned meshes require the following component at all times:

-   BindSkeletonRoot
    -   Contains a reference to one of the following:
        -   *Base skeleton*
        -   *Exposed bone*
        -   *Exported bone*
        -   Already bound skinned mesh
    -   Added during baking (only if descendent of converted skeleton)

In addition, Kinemation requires one of these components to perform a binding:

-   MeshBindingPathsBlobReference
    -   Fails if the target skeleton does not have a
        `SkeletonBindingPathsBlobReference`
    -   Added during baking
-   DynamicBuffer\<OverrideSkinningBoneIndex\>
    -   Directly maps each mesh bone (bindpose) to a skeleton bone
    -   Has priority

If Kinemation detects the correct components exist for binding, it will attempt
to bind. Every attempt will result in the following structural changes of
components:

-   Added
    -   Parent
    -   CopyParentWorldTransformTag
    -   SkeletonDependent – Internal Cleanup type
    -   ChunkSkinningCullingTag

A binding attempt can fail. In that case, Kinemation will parent the skinned
mesh to a special entity with the `FailedBindingsRootTag`. This is done for a
better debugging experience. After a failed binding attempt, you will need to
add the `NeedsBindingFlag` and set its value to `true` in order to reattempt a
binding.

### Blend Shape Mesh

A blend shape mesh is a *deformed mesh* entity that can be deformed by blend
shapes. In the case a blend shape mesh is also a *skinned mesh*, the blend
shapes will be applied before skinning. A blend shape mesh has the following
components:

-   BlendShapeState
    -   Required
    -   Controls rotations and dirty flags
    -   Added during baking
-   DynamicBuffer\<BlendShapeWeight\>
    -   Required
    -   3 sets of weights in rotation, with a single weight per shape
    -   Added during baking

When working with blend shapes, it is recommended to use `BlendShapeAspect`
which will ensure you write to the correct weights for a given frame.

### Dynamic Mesh

A dynamic mesh is a *deformed mesh* entity whose animated vertices are provided
directly by the CPU, typically in Burst jobs. Dynamic meshes are excellent for
simulating soft-bodies or particles on the CPU. If a dynamic mesh is also a
*blend shape mesh* or *skinned mesh*, the other deformations are applied on top
of the dynamic mesh vertices. A dynamic mesh has the following components:

-   DynamicMeshState
    -   Required
    -   Controls rotations and dirty flags
    -   Must be added by user
-   DynamicBuffer\<DynamicMeshVertex\>
    -   Required
    -   3 sets of vertices in rotation
    -   Must be added by user
-   DynamicMeshMaxVertexDisplacement
    -   Required
    -   Contains the max distance a vertex is moved from its undeformed position
    -   Must be updated by user for culling to work correctly
    -   Must be added by user

When working with dynamic meshes, it is recommended to use `DynamicMeshAspect`
which will ensure you write to the correct vertices for a given frame.
Additionally, because theoretically any mesh could be a dynamic mesh, setup must
be configured by the user at bake time. A full guide is provided
[here](Dynamic%20Meshes.md).

### Shared Submesh

A *shared submesh* is an entity which shares the result of skinning with a
skinned mesh. This typically happens when a Skinned Mesh Renderer that uses
multiple materials is baked into an entity. It always has a
`CopyDeformFromEntity` component which points to the entity with the skin it
should borrow.

A *shared submesh* also has the following components added during baking:

-   ChunkCopyDeformTag
    -   Internal
    -   Also added at runtime if missing
-   Material Properties
    -   See *deformed mesh* for list

The biggest surprise with *shared submeshes* is that Kinemation bakes the
hierarchy a little different from the rest of Entities Graphics. It looks like
this:

-   Skeleton
    -   Deformed Mesh (Primary Entity) – Material 0
        -   Shared Submesh – Material 1
        -   Shared Submesh – Material 2
        -   …

## On to Part 2

Wow! You made it through. Or maybe you skimmed it. Either way, I hope it is
obvious that Kinemation has a lot going on to make everything work seamlessly.
Fortunately, baking does most of the heavy lifting for you. So actually getting
started with simple skinned meshes couldn’t be easier!

[Continue to Part 2](Getting%20Started%20-%20Part%202.md)
