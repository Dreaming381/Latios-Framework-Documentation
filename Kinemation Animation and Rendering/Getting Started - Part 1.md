# Getting Started with Kinemation – Part 1

This is the fifth preview version released publicly. As such, there’s still more
to come. For simple use cases, it is easy to get started. But for advanced
production workflows, some of that burden may fall on the user.

It will be up to you, a member of the community, to help guide Kinemation’s
continual development to meet the needs of real-world productions. Please be
open and loud on your journey. Feedback is critical.

This Getting Started series is broken up into multiple parts. This first part
covers the terminology and runtime structure. It is okay if you do not
understand everything in this part or choose to skim through it. The remaining
parts will guide you through the process of configuring and animating a
character as an entity.

## Minimum Requirements

-   Currently Supported Platforms (out-of-the-box)
    -   Windows
    -   Mac OS
    -   Linux
    -   Android
-   GPU Minimum Requirements
    -   Supports Entities Graphics
    -   Supports 8 simultaneous compute buffers in compute shaders
    -   Has 32 kB of shared memory per thread group (except Android)
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
analyzes that during baking instead.

However, Kinemation also allows skeletons to be built procedurally from code.

## Renderers

Renderers in Kinemation are quite similar to Entities Graphics. They are
fundamentally built on `MaterialMeshInfo` and `RenderMeshArray`, and can contain
optional material property overrides. However, authoring Mesh Renderers can have
multiple materials, and how Kinemation deals with this is different than
Entities Graphics.

Beginning in Entities Graphics 1.1, a single `MaterialMeshInfo` instance can
refer to multiple material-mesh-submesh triplets within a `RenderMeshArray`.
However, Entities Graphics will only bake entities to use this format if a
particular scripting define is enabled. Thus, Entities Graphics builds entity
hierarchies like this:

-   If single material or scripting define set and not skinned
    -   primaryEntity – all rendering components
-   Otherwise
    -   primaryEntity – no rendering components
        -   additionalEntity0 – rendering components for first material
        -   additionalEntity1 – rendering components for second material
        -   …

Kinemation does things a little differently by default.

-   If renderer has both opaque and transparent materials
    -   primaryEntity – rendering components for all opaque materials
        -   additionalEntity – rendering components for all transparent
            materials
-   Otherwise
    -   primaryEntity – rendering components for all materials

This means that in Kinemation by default, there will only ever be at most two
entities per authoring renderer. No scripting define is required, and the
behavior is true for both Mesh Renderers and Skinned Mesh Renderers.

While this is the default setup, Kinemation supports completely overriding the
setup and distribution of entities. You can find a comprehensive set of API in
OverrideMeshRendererBaker.cs and there’s an example of its usage for [dynamic
meshes](Dynamic%20Meshes.md).

Renderer entities can also be given the `RendererVisibilityFeedbackFlag` which
is an `IEnableableComponent` that is enabled if the renderer was visible in the
previous frame, and disabled otherwise. Renderer entities can also be given
`PostProcessMatrix` and `PreviousPostProcessMatrix` for applying shear and other
visual matrix transform effects to the entities for rendering.

Renderers are invalid if they do not have the appropriate world-space transform
component for the transform system in use (WorldTransform for QVVS and
LocalToWorld for Unity Transforms).

## Deforming Renderers

Deforming renderers are *renderer* entities (that is, they have the
`MaterialMeshInfo` component) that contain special types of material property
components used for deformation. Common examples would be
`CurrentMatrixVertexSkinningShaderIndex` or `CurrentDeformShaderIndex`. The
primary entity is given a copy of all deforming material properties of
additional entities. And each additional entity is given a
`CopyDeformFromEntity` component which points back to the primary entity. This
component will cause the entity to receive the culling status and copy all
material properties of the primary entity in the culling loop, rather than rely
on its own culling status.

Deforming renderers are automatically created from baking skinned mesh
renderers, but can also be created by setting `isDeforming` to `true` in
`MeshRendererBakeSettings`.

## Bound Mesh Deforming Renderers

Building on top of *deforming renderers*, *bound mesh deforming renderers*
additionally contain the `MeshDeformDataBlobReference` component. At runtime,
these meshes are registered and receive an internal `BoundMesh` cleanup
component. If you need to change the mesh blob because you changed the mesh at
runtime, you can add and enable the `NeedsBindingFlag` component.

Bound meshes upload unique mesh data to the GPU and reference count usage.
Visible bound mesh entities will also reserve GPU memory and update deforming
material properties each frame during the culling loop.

You will also notice the internal `ChunkDeformPrefixSums` component added to
Bound mesh entities at runtime.

## Dynamic Meshes

*Dynamic meshes* are a type of *bound mesh deforming renderer* which contain
components used for deforming the mesh each frame on the CPU. They contain the
following component types which must be added via user code:

-   `DynamicBuffer<DynamicMeshVertex>`
-   `DynamicMeshState`
-   `DynamicMeshMaxVertexDisplacement`

The first two components are abstracted by `DynamicMeshAspect` for significantly
easier modification. Dynamic meshes pack three copies of the deformed mesh
vertices in a dynamic buffer and rotate the roles of the three copies each
frame. This scheme supports high-performance motion vectors and other custom
shader effects. The rotation is updated in `MotionHistorySuperSystem`.

Dynamic mesh vertices are uploaded to the GPU in the culling loop once the
entity is determined to be visible for the given frame. GPU data is not cached
between frames.

Dynamic meshes are explained in more detail [here](Dynamic%20Meshes.md).

## Blend Shape Meshes

*Blend shape meshes* are *bound mesh deforming renderers* that use blend shape
deformations. They contain two components `DynamicBuffer<BlendShapeWeight>` and
`BlendShapeState` which are automatically added when baking a Skinned Mesh
Renderer with blend shapes and are abstracted by `BlendShapeAspect`. Similar to
dynamic meshes, the weights buffer stores three copies of all weights with the
roles rotated each frame. The rotation is updated in `MotionHistorySuperSystem`.

Blend shapes are applied in a compute shader on the GPU for visible entities
inside the culling loop. If the entity is also a *dynamic mesh*, the blend
shapes are applied additively on top of dynamic mesh vertices. This can be
disabled by adding the `DisableComputeShaderProcessingTag` component to the
entity.

A weight value of 0 will skip application of a blend shape, while a weight value
of 1 will apply the blend shape at full strength. Weight values outside the
range of [0, 1] are allowed for exaggeration of the shape.

## Skinned Meshes

*Skinned meshes* are *bound mesh deforming renderers* that use a skeleton to
drive the deformations. They are created via the presence of a
`BindSkeletonRoot` component as well as either the
`MeshBindingPathsBlobReference` or the
`DynamicBuffer<OverrideSkinningBoneIndex>.` The value of `BindSkeletonRoot` can
point to a skeleton entity, a bone entity, or another entity containing a
`BindSkeletonRoot`. The binding system will follow this chain until it finds a
proper skeleton entity with a `SkeletonRootTag` component. `BindSkeletonRoot` is
automatically initialized on entities with a Skinned Mesh Renderer which are
descendants of a baked Animator and have bind poses. By default, a Skinned Mesh
Renderer with bind poses will bake with the `MeshBindingPathsBlobReference`
component. `OverrideSkinningBoneIndex` is intended for procedural setups and is
not commonly used.

A large number of changes are made to skinned mesh entities at runtime during
the binding phase. The skinned mesh entity is reparented directly to the
skeleton entity and given the `CopyParentWorldTransformTag` component. In Unity
Transforms, the `LocalTransform` is also set to `Identity` and modification will
result in undefined behavior. The entity will also be given the internal cleanup
component `SkeletonDependent` and the chunk component `ChunkSkinningCullingTag`.

The skinned mesh entity will go through a skeleton binding phase where an
attempt is made to compute bone indices for the bind poses. If this operation
fails, instead of being parented to the skeleton entity, the skinned mesh entity
will instead be parented to an entity with the `FailedBindingsRootTag`. If the
target skeleton needs to be changed at runtime, add and enable to
`NeedsBindingFlag` component.

Skeletal skinning is partly or completely processed in a compute shader. The
results are applied additively on top of dynamic meshes and blend shape
deformations. Skeletal skinning can be disabled via
`DisableComputeShaderProcessingTag`.

When skeletal skinning is active, `RenderBounds` and `WorldRenderBounds` are
ignored, and instead culling uses data from the skeleton entities for culling.
To add additional padding to the culling bounds to account for additional vertex
deformation effects, use the `ShaderEffectRadialBounds` component.

## Skeletons

As discussed at the beginning, a *skeleton* entity is an entity that represents
a collection of bones and is what skinned meshes bind to. A skeleton entity has
the `SkeletonRootTag` component. It will also typically have the
`SkeletonBindingPathsBlobReference` component, though this is technically
optional.

Skeleton entities are automatically baked by the presence of an Animator
component, and will bake both the `SkeletonRootTag` and the
`SkeletonBindingPathsBlobReference`.

At runtime, several other internal components will be added to the skeleton
entity:

-   `DynamicBuffer<DependentSkinnedMesh>`
-   `SkeletonBoundsOffsetFromMeshes`
-   `ChunkPerCameraSkeletonCullingMask`
-   `ChunkPerCameraSkeletonCullingSplitsMask`

A skeleton entity is not valid by itself. It must additional be one of two
specialized archetypes, exposed or optimized.

## Exposed Skeletons

An *exposed skeleton* is a *skeleton* entity where each bone within the skeleton
is a separate entity. An exposed skeleton has a `DynamicBuffer<BoneReference>`
which contains a list of the bone entities.

Exposed skeletons are automatically baked when `Animator.hasTransformHierarchy`
is `true`.

If you modify the `BoneReference` buffer (that is, change which entities are
part of the skeleton at runtime), you will need to add and enable the
`BoneReferenceIsDirtyFlag` component to the exposed skeleton.

At runtime, the internal component `ExposedSkeletonCullingIndex` will be added
to the exposed skeleton entity.

## Exposed Bones

**Note: What Unity’s Rig tab in the model importer refers to as “exposed bones”
are consequently referred to as “exported bones” in Kinemation. This name better
suits the behavior of such bones as their transform data SHOULD NOT be modified
by user code. “Exposed bones” in Kinemation refer to bones whose transforms CAN
be modified by user code, and such modifications will be reflected in attached
Skinned Meshes. When discussing Kinemation issues, try to stick to the
Kinemation nomenclature.**

An *exposed bone* is a bone that belongs to an *exposed skeleton’s*
`BoneReference` buffer. It requires quite a few components to function
correctly.

-   Always required
    -   `BoneIndex`
    -   `BoneOwningSkeletonReference`
-   Required when using Unity Transforms
    -   `LocalToWorld`
-   Required when using QVVS Transforms
    -   `WorldTransform`
    -   `PreviousTransform`
    -   `TwoAgoTransform`
-   Required when influencing skinned meshes
    -   `BoneCullingIndex`
    -   `BoneBounds`
    -   `BoneWorldBounds`
    -   `ChunkBoneWorldBounds`

With the exception of transform components, all required exposed bone components
can be added via `CullingUtilities.GetBoneCullingComponentTypes()`. As Unity
Transforms does not support motion history, deforming material properties that
request *Previous* or *TwoAgo* representation of the skeleton in Unity
Transforms projects will simply be given the current frame’s representation
instead.

All exposed bone components are added by default during baking. No components
are added or removed at runtime. However, all non-transform components will be
reinitialized upon the first time Kinemation systems see the *exposed skeleton*
or when `BoneReferenceIsDirtyFlag` is present and enabled on the *exposed
skeleton* entity.

The `BoneOwningSkeletonReference` is simply an entity reference back to the
*exposed skeleton*. `BoneIndex` is the index of the exposed bone entity inside
the *exposed skeleton’s* `BoneReference` buffer. The *exposed skeleton* entity
is also typically the first exposed bone entity in the `BoneReference` buffer.

Skinning and culling are fully driven by the world-space transforms of the
exposed bones. This means Kinemation will react appropriately to logic-driven
modification to bone transforms. In addition, Kinemation is unbothered by
transform hierarchy changes applied to the bones. This is especially useful when
using Unity Transforms and Unity Physics to drive a ragdoll character.

**Warning:** If an *exposed skeleton* references an invalid exposed bone, the
resulting behavior is undefined, and usually pretty bad.

## Optimized Skeletons

An *optimized skeleton* is a *skeleton* entity that stores its bones directly
inside dynamic buffers and blob assets. It is defined by the presence of the
`DynamicBuffer<OptimizedBoneTransform>`.

Similar to dynamic meshes and blend shapes, optimized skeletons also use a
rotation system for transforms that is updated in `MotionHistorySuperSystem`.
However, instead of having 3 sets of transforms, it has 6. This is because each
rotation stores both local and root-space transforms. This mechanism is
abstracted by `OptimizedSkeletonAspect`.

Optimized skeletons are automatically baked when
`Animator.hasTransformHierarchy` is `false`. The default baker will add an
`OptimizedSkeletonHierarchyBlobReference` component, which can be used to find
parent and children bone indices from any other bone index. The baker will also
add `DynamicBuffer<OptimizedBoneInertialBlendState>` which is used for inertial
blending (more on that later). These two component types are technically
optional, but are required for use of `OptimizedSkeletonAspect`.

At runtime, additional components are added to optimized skeleton entities:

-   `OptimizedSkeletonTag`
-   `OptimizedSkeletonState`
-   `DynamicBuffer<OptimizedBoneBounds>`
-   `OptimizedSkeletonWorldBounds`
-   `ChunkOptimizedSkeletonWorldBounds`

### OptimizedSkeletonAspect

`OptimizedSkeletonAspect` contains many features to help manipulate the bones in
an optimized skeleton. Some of these features are exclusive to optimized
skeletons.

Optimized skeletons require an initialization step. This typically happens in
`MotionHistoryUpdateSystem`, but if the entity was instantiated after this point
in the frame, you may need to initialize the entity manually via the
`ForceInitialize()` method.

An optimized skeleton can either be in a synced or unsynced state. Accessing
`rawLocalTransformsRW`, sampling poses from skeleton clips (more on that in a
later part), or applying inertial blending will put the skeleton in an unsynced
state. You can convert to a synced state via calling `EndSamplingAndSync()`.
When synced, the root transforms are up-to-date.

When the skeleton is synced, you can index into the `bones` property to get an
`OptimizedBone`. This type allows you to modify the bone’s transform and changes
will automatically be propagated through the hierarchy.

Inertial blending is a technique for smoothing out transitions between
discontinuous animations over time. The technique only requires a brief history
of the bone transforms and the present state of the bone transforms that would
be used if no blending were applied. You can start an inertial blend with
`StartNewInertialBlend()`. Then each frame you smooth out the transforms with
`InertialBlend()`. It is safe to interrupt an inertial blend with a new inertial
blend.

### OptimizedRootDeltaROAspect

The root bone of an optimized skeleton typically contains the root motion delta
when sampling from animation clips. However, Unity’s source generators sometimes
get tripped up when attempting to use `OptimizedSkeletonAspect` and
`TransformAspect` in the same job. This makes it difficult to apply the root
motion to the skeleton entity’s transform.

`OptimizedRootDeltaROAspect` provides read-only access to just the root bone of
an optimized skeleton to assist with root motion operations. Root motion can be
computed in one job using `OptimizedSkeletonAspect`, and applied in another
using `OptimizedRootDeltaROAspect`.

## Exported Bones

An *exported bone* is an entity that copies the root-space
`OptimizedBoneTransform` QVVS of an *optimized skeleton* bone and assigns it to
its own `LocalTransform` and `WorldTransform` (the latter for stretch, QVVS
only). When parented directly to the *optimized skeleton* entity, it effectively
mimics the transform of the optimized bone along with all the optimized bone’s
animations. This is often used for rigid character accessories like weapons or
hats.

Any entity can become an exported bone simply by adding
`BoneOwningSkeletonReference` and `CopyLocalToParentFromBone` components and
setting their values appropriately. Bakers will automatically do this for bones
made available in the *Rig* tab of the optimized skeleton’s import settings.

*Exported bones* update during either `TransformSuperSystem` or
`TransformSystemGroup` depending on which transform system you use. The
*optimized skeleton* does not know about nor care about *exported bones*, which
means you can have multiple *exported bones* track the same optimized bone. Be
careful though, because *exported bones* can be relatively costly to update.

## On to Part 2

Wow! You made it through. Or maybe you skimmed it. Either way, I hope it is
obvious that Kinemation has a lot going on to make everything work seamlessly.
Fortunately, baking does most of the heavy lifting for you. So actually getting
started with simple skinned meshes couldn’t be easier!

[Continue to Part 2](Getting%20Started%20-%20Part%202.md)
