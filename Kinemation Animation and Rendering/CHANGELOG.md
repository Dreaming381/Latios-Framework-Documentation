# Changelog

All notable changes to this package will be documented in this file.

The format is based on [Keep a Changelog](http://keepachangelog.com/en/1.0.0/)
and this project adheres to [Semantic
Versioning](http://semver.org/spec/v2.0.0.html).

## [0.10.0] – 2024-5-26

Officially supports Entities [1.2.1]

### Added

-   *New Feature:* AclUnity has been updated to support masked sampling and
    automatic weight accumulation for skeletons
-   *New Feature:* LOD Crossfade (including SpeedTree) is now supported,
    although animated crossfade mode is not
-   *New Feature:* Added `LodCrossfade`, `LodHeightPercentages`,
    `LodHeightPercentagesWithCrossfadeMargins`, and `SpeedTreeCrossfadeTag`
    components for the new LOD implementation
-   *New Feature:* Dual Quaternion Skinning is now supported, which can be
    enabled in the *Latios Vertex Skinning Node* in Shader Graph or via the *Use
    Dual Quaternion Skinning* option in *Skinned Mesh Settings* authoring
    component when using the *Latios Deform Node* which adds the
    `DualQuaternionSkinningDeformTag` for runtime
-   *New Feature:* Added `SkeletonClip.SamplePose()` overloads which accept a
    `ReadOnlySpan<ulong>` bitmask selecting which bones in the skeleton should
    be sampled
-   *New Feature:* Added `ParameterClip.SampleSelectedParameters()` which
    accepts a `ReadOnlySpan<ulong>` bitmask selecting which parameter indices
    should be sampled
-   *New Feature:* Added `RuntimeBlobBuilders` namespace with utility struct
    types `SkeletonClipSetSampler` and `SkeletonClipSetSampleData` for creating
    skeleton clip blobs at runtime
-   Added `IBaker` extension method `RequestBoneNames()` providing a
    `BoneNamesRequestHandle`, which like a `SmartBlobberHandle` can be later
    resolved to obtain bone names for an optimized hierarchy
-   Added `SkeletonClip.SamplePose()` overloads which write directly to a raw
    `NativeArray<TransformQvvs>()` for total manual control
-   Added `OptimizedSkeletonAspect.nextSampleWillOverwrite` property to manually
    override if the next sample will be overwriting or additive, especially
    effective when using masked sampling
-   Added *URP14-LOD_Crossfade_Fix_Node* Shader Subgraph for enabling LOD
    Crossfade in 2022 LTS, though this is unnecessary in Unity 6
-   Added support for Light Probe Anchor Overrides
-   Added `TransformQvvs` extension method `NormalizeBone()` which will correct
    a bone transform based on the accumulated weight stored in the `worldIndex`

### Changed

-   **Breaking:** The Entities Graphics LOD components and algorithm are no
    longer supported, as they have been removed in favor of the new
    high-performance algorithm that supports LOD Crossfade
-   **Breaking:** `MeshRendererBakeSettings` `lodGroupEntity` and `lodGroupMask`
    have been replaced with `lodSettings`, which is also the new output argument
    of `IBaker` extension method `GetLOD()`
-   **Breaking:** Sampling from a `SkeletonClip` now writes the bone weights
    into the `worldIndex` of the `TransformQvvs`, which is used when performing
    normalization
-   **Breaking:** Renamed `BufferPoseBlender` method `NormalizeRotations()` to
    `Normalize()`, as this method will also normalize the accumulated weights
-   **Breaking:** `BufferPoseBlender.ComputeRootSpaceTransforms()` now expects a
    `ReadOnlySpan<short>` for `parentIndices` instead of `ref BlobArray<short>`
-   `SkeletonBindingPathsBlob` methods `StartsWith()` and
    `TryGetFirstPathIndexThatStartsWith()` are now generic methods that can work
    with any Burst-compatible string type
-   Light probes now use the center of world bounding box as the reference
    point, same as game objects
-   `ShaderEffectRadialBounds` now applies to all renderers, not just deforming
    renderers

### Fixed

-   Fixed `LatiosFrozenStaticRendererSystem` using the wrong query
-   Fixed `GraphicsBufferBroker` sometimes not performing a buffer resize
    correctly and missing the last dispatch
-   Fixed skinning compute shader compilation on some platforms
-   Fixed CopyBytes compute shader only copying a fourth of the bytes it is
    supposed to
-   Fixed issue where sometimes `LatiosEntitiesGraphicsSystem` would not
    allocate a large enough buffer for new entities
-   Fixed wrong namespaces for `CombineExposedBonesSystem`,
    `UpdateSkinnedPostProcessMatrixBoundsSystem`, `UpdateBrgBoundsSystem`,
    `LatiosAddWorldAndChunkRenderBoundsSystem`,
    `LatiosRenderBoundsUpdateSystem`, and
    `LatiosUpdateEntitiesGraphicsChunkStructureSystem`
-   Fixed rebinding flag bitfield not being initialized correctly

### Improved

-   *New Feature:* All `ComponentSystemGroup` types can properly update systems
    making use of Latios Framework features such as conditional updates and
    dependency tracking
-   Baking for `MaterialMeshInfo` using range indexing (multiple meshes and/or
    materials) now performs basic deduplication at bake time, reducing subscene
    load costs when a multi-material prefab is spammed across the subscene
-   Baking `RenderMeshArray` no longer allocates GC
-   `WorldToLocal` matrices are now computed correctly for shaders in more
    extreme scenarios (smaller transforms)
-   Refined criteria for when deformed bounds need to be updated in a chunk,
    resulting in fewer updates
-   Tiny improvements to handling of enabled components via
    `ArchetypeChunk.SetComponentEnabledForAll()` rather than setting the value
    for each bit
-   Removed a frequent GC allocation in `LatiosEntitiesGraphicsSystem` caused by
    `SortedSet.Add()`, replacing it with an unmanaged implementation

### Removed

-   Removed `UpdateLODsSystem` and `LatiosLODRequirementsUpdateSystem`

## [0.9.3] – 2024-3-2

Officially supports Entities [1.1.0-pre.3]

### Fixed

-   Fixed BatchSkinning compute shader failing to compile on some platforms
-   Fixed moving LOD Groups not updating correctly
-   Fixed memory leak from batches not being removed from the
    `BatchRendererGroup`
-   Fixed a range exception while computing optimized skeleton bounds if the
    skeleton was initialized via `OptimizedSkeletonAspect.ForceInitialize()`
    before it was registered with the binding system

## [0.9.2] – 2024-2-4

Officially supports Entities [1.1.0-pre.3]

### Fixed

-   Fixed wrong transform being returned when accessing `twoAgoLocalTransform`
    and `twoAgoRootTransform` in `OptimizedBone`

## [0.9.1] – 2024-1-27

Officially supports Entities [1.1.0-pre.3]

### Added

-   Added `ExcludeFromSkeletonAuthoring` to prevent a baked Game Object and its
    children from being included in a skeleton

### Fixed

-   Fixed baking of optimized skeletons and animations for prefabs
-   Fixed exception during skeleton baking if a descendent skinned mesh does not
    have a valid mesh assigned
-   Fixed inertial blending buffers being uninitialized on optimized skeletons,
    resulting in a hard crash
-   Fixed transparency sorting when using Unity Transforms

## [0.9.0] – 2024-1-20

Officially supports Entities [1.1.0-pre.3]

### Added

-   *New Feature:* Added `SkinningAlgorithms` which provides a suite of
    utilities for deforming Dynamic Meshes
-   *New Feature:* Added `GraphicsBufferBroker` and related APIs for custom user
    operations involving new or existing graphics buffers within
    `PresentationSystemGroup` or the culling loop
-   Added `MeshNormalizationBlob` and added an instance to `MeshDeformDataBlob`
-   Added `BlobBuilder` extension method `CreateParameterClip()` for creating a
    parameter animation clip using ACL compression embedded inside a
    user-defined blob asset
-   Added `CountLoopCycleTransitions()` method to `SkeletonClip` and
    `ParameterClip`
-   Added `SkinnedMeshBindingAspect`, `SkeletonSkinBindingsAspect`, and
    `SkinBoneBindingsCollectionAspect` for obtaining cached bone bindings
    between skinned meshes and skeletons for use with `SkinningAlgorithms`
-   Added `BatchCullingFlags` to `CullingContext`
-   Added `DisableComputeShaderProcessingTag` for disabling GPU blend shapes and
    skeletal skinning on dynamic meshes that wish to perform these operations on
    the CPU instead
-   Added `OptimizedRootDeltaROAspect` for working around a Unity source
    generator issue when attempting to use `OptimizedSkeletonAspect` and
    `TransformAspect` in the same job
-   Added `CullingRoundRobinEarlyExtensionsSuperSystem` and
    `CullingRoundRobinLateExtensionsSuperSystem` for injecting custom graphics
    buffer operations into the culling loop

### Changed

-   **Breaking:** Changed `parameterNames` in `ParameterClipSetBlob` and all
    associated APIs to use `FixedString128Bytes` instead of `FixedString64Bytes`
-   **Breaking:** All renderers with multiple materials will now by default bake
    into only one or two entities each, rather than an entity per material
-   **Breaking:** Renamed `KinemationBakingBootstrap` method
    `InstallKinemationBakersAndSystems()` to `InstallKinemation()`
-   **Breaking:** Redesigned all methods in
    `OverrideMeshRendererBakerExtensions` to utilize the new multi-material
    runtime features and new renderer baking backend
-   The default compression level when compressing animation clips is now
    “automatic mode”, which is set to a value of 100

### Fixed

-   Fixed baking blend shapes when the GPU buffer size was larger than the size
    expected from the mesh object
-   Fixed renderer entities with only blend shapes being baked with
    `BindSkeletonRoot` if they had an Animator ancestor
-   Fixed `OptimizedBone` change propagations for individual position, rotation,
    and scale values
-   Fixed missing `Systems` namespace on `CullingComputeDispatchSubSystemBase`,
    `UploadDynamicMeshesSystem`, and `BlendShapesDispatchSystem`
-   Fixed graphics buffer being too small when lots of blend shape meshes are
    rendered in a single frame
-   Fixed live baking losing track of `MeshDeformDataBlobs` due to the binding
    system not detecting the binding discrepancy
-   Fixed `RenderBounds` changes not being considered when updating the bounds
    of deforming meshes that do not use skeletal skinning
-   Fixed Unity Transforms not resetting `LocalTransform` to `Identity` when a
    skinned mesh is bound to a skeleton
-   Fixed skinned meshes having wrong world bounds in the Editor when live
    baking due to `ChunkBoneWorldBounds` being dropped during baking for some
    unknown reason
-   Fixed dynamic meshes and blend shapes failing to render when instantiated in
    the same frame after `MotionHistorySuperSystem`

### Improved

-   ACL has been updated to version 2.1.0
-   All renderers use a new baking backend which is hopefully superior in
    performance, memory usage, and robustness compared to the Entities Graphics
    backend
-   Culling systems are now visible in the Systems window
-   `LatiosEntitiesGraphicsSystem` uses `WorldUpdateAllocator` instead of
    `TempJob`
-   A new validator checks for bad rendering archetypes such as Unity Transforms
    in a QVVS Transform project and reports any offending archetype to the
    console
-   `OptimizedSkeletonState` is now automatically added to entities with
    `OptimizedBoneTransform` if missing during binding
-   `BindSkeletonRoot` will now search through multiple iterations of entity
    references to find a valid skeleton during binding

### Removed

-   **Breaking:** Mecanim has been moved out of Kinemation and into Mimic
-   **Breaking:** The `TextBackend` namespace and associated functionality has
    been moved out of Kinemation and into Calligraphics
-   **Breaking:** Removed `RenderQuickToggleEnableFlag` as `MaterialMeshInfo`
    may now be used for the same purpose

## [0.8.6] – 2023-12-9

Officially supports Entities [1.0.16]

### Fixed

-   Fixed memory stomps from off-screen meshes using dynamic mesh buffers or
    blend shapes
-   Fixed `PostProcessTransform` and `PreviousPostProcessTransform` not being
    applied to rendering transforms
-   Fixed changes to `PostProcessTransform` and `PreviousPostProcessTransform`
    not being applied for otherwise static entities

## [0.8.5] – 2023-12-3

Officially supports Entities [1.0.16]

### Added

-   Added Android support

### Fixed

-   Fixed AclUnity plugin failing to load in linux builds
-   Fixed Latios Deform crashing when there were more than one large skeleton
    entities
-   Fixed Mecanim root motion rotation rotating the opposite way
-   Fixed Mecanim accuracy issue with small clip weights
-   Fixed Mecanim ComponentType conflict with optimized skeletons and QVVS
    transforms
-   Fixed root motion accuracy issues with exit times

## [0.8.3] – 2023-10-15

Officially supports Entities [1.0.16]

### Fixed

-   Fixed `TextBackendUpdateSystem` always being installed when using injection
    workflow, even if Kinemation is not installed
-   Fixed RenderVisibilityFeedbackFlag not being enabled after it has been
    disabled if the whole chunk was disabled previously

## [0.8.2] – 2023-10-8

Officially supports Entities [1.0.16]

### Fixed

-   Fixed transparency sorting when using Unity Transforms and PostProcessMatrix

## [0.8.0] – 2023-9-23

Officially supports Entities [1.0.16]

### Added

-   *New Feature*: Added Unity Transforms (Entities 1.0) support
-   *New Feature*: Added Mecanim Animator Controller runtime which can bake an
    Animator Controller and provides `MecanimAspect` API for runtime
    manipulation
-   *New Feature*: Added `TextBackend` namespace and functionality for baking
    and rendering text
-   Added `RootMotionOverrideMode` to `SkeletonClipCompressionSettings` which
    can be used to specify whether root motion should be baked into bone index 0

### Changed

-   **Breaking:** `OptimizedSkeletonAspect.skeletonWorldTransform` no longer
    returns by `ref readonly` as a backing value can no longer be guaranteed in
    Unity Transforms mode
-   By default, animation clips bake with root motion enabled such that motion
    of the root is baked into bone index 0’s animation

### Improved

-   Several systems now Burst-compile their `OnCreate()` method

### Removed

-   Removed `ExposedBoneInertialBlendState` as implementations for exposed
    skeletons may want to store this in a different way

## [0.7.7] – 2023-8-15

Officially supports Entities [1.0.14]

### Improved

-   Improved shadow map cascade culling to be more precise, resulting in fewer
    draws of near objects into the shadow maps

## [0.7.5] – 2023-7-2

Officially supports Entities [1.0.11]

### Added

Added `RenderVisibilityFeedbackFlag` which is an enableable component whose
enabled status represents whether the entity was rendered by any view in the
previous frame

### Fixed

-   Fixed async GPU readback exception when baking blend shapes
-   Fixed blend shape weights being baked 100 times larger than they should be
-   Added a workaround for an issue where Unity would bind blend shape deltas as
    a `ByteAddressBuffer` instead of a `StructuredBuffer`
-   Fixed blend shapes initializing GPU buffers for meshes that aren’t dynamic
    meshes
-   Fixed blend shapes initializing GPU buffers when all weights are zero
-   Fixed blend shapes uploading the wrong data at the wrong offsets
-   Fixed blend shape meshes that are not skinned but are parented to a bone of
    a skeleton attempting to bind to the skeleton
-   Fixed incremental baking of optimized skeletons, blend shapes, and dynamic
    meshes producing buffer size mismatch errors in the Editor

### Improved

-   Added a warning when baking a normal Mesh Renderer using a shader with
    deformation properties, as this will lead to incorrect rendering results

## [0.7.4] – 2023-6-18

Officially supports Entities [1.0.10]

### Fixed

-   Fixed initialization flag not clearing in `OptimizeSkeletonAspect`
-   Fixed `ArgumentOutOfRangeException` in skinning for large quantities of
    skinned meshes
-   Fixed exposed bones ignoring mesh bounds offsets from blend shapes or
    shaders
-   Fixed optimized skeletons not receiving mesh bounds offsets
-   Fixed issue where a corrupted optimized skeleton buffer could grow
    infinitely

## [0.7.1] – 2023-6-3

Officially supports Entities [1.0.10]

### Fixed

-   Fixed shader graph node generation generating the wrong include paths
-   Fixed compile errors when using LATIOS_TRANSFORMS_UNITY scripting define
    symbol

## [0.7.0] – 2023-5-29

Officially supports Entities [1.0.10]

### Added

-   *New Feature:* Added Blend Shapes support
-   *New Feature:* Added Dynamic Meshes which allow meshes to be animated in
    Burst jobs without separate Mesh instances
-   *New Feature:* Added `OptimizedSkeletonAspect` and `OptimizedBone` which
    provide a synchronized hierarchy API for working with optimized skeleton
    bones in local, root, and world space at the same time
-   *New Feature:* Added Inertial Blending components for both exposed and
    optimized skeletons
-   *New Feature:* Added custom shader graph nodes for deformations which allow
    targeting the motion history of a deforming mesh
-   *New Feature:* Added `RenderQuickToggleEnableFlag` which is an optional flag
    to quickly toggle whether or not an entity can be rendered
-   *New Feature:* Added support for skeletons up to 32 767 bones
-   Added `PostProcessMatrix` for augmenting a `WorldTransform` with shear for
    rendering purposes
-   Added `SkeletonClipCompressionSettings.maxUniformScaleError` as this
    parameter is compressed into a separate clip currently
-   Added several material property components that are compatible with the new
    shader graph nodes

### Changed

-   **Breaking:** Updated the AclUnity C\# API to work with QVVS
-   **Breaking:** Kinemation is now designed to work with QVVS Transforms
    instead of Unity Transforms for skeletal deformations and rendering in
    general
-   **Breaking:** The culling loop has been reworked for improved performance
    using a round-robin scheduler, which any custom culling code must now adjust
    for
-   **Breaking:** `OptimizedBoneToRoot` and associated API were replaced with
    new components abstracted by `OptimizedSkeletonAspect`, which is the new way
    to work with optimized skeletons
-   **Breaking:** Renamed `MeshSkinningBlob` and `MeshSkinningBlobReference` to
    `MeshDeformDataBlob` and `MeshDeformDataBlobReference` and their internals
    have been significantly refactored
-   **Breaking:** Replaced `OverrideMeshRendererBakerBase` with Baker extension
    methods
-   **Breaking:** Renamed `ShareSkinFromEntity` to `CopySkinFromEntity`
-   Many Unity systems now are replaced with Kinemation versions, and the whole
    rendering stack works a little bit differently internally as a result,
    though the end result should still be the same
-   Skinned mesh and skeleton binding requirements have changed to support new
    deforming mesh types

### Fixed

-   Fixed leftover chunk components existing on entities with removed meshes
    during incremental baking
-   Fixed various missing `[TemporaryBakingType]` attributes

### Improved

-   Improved XML documentation coverage
-   Switched from `ComputeBuffer` to `GraphicsBuffer` for all internal buffers
-   BRG bounds are now computed when all bounds are updated, which improves main
    thread performance during rendering
-   Missing mask components for normal renderers are now added during the single
    binding reactive system sync point
-   Light Probes processing now use jobs and Burst
-   Skeletal animated meshes no longer compute their own world bounds

### Removed

-   Removed `maxNegligibleTranslationDrift` and `maxNegligibleScaleDrift` from
    `SkeletonClipCompressionSettings` as these are now computed dynamically in
    ACL
-   Removed `ParentScaleInverse` data from `OptimizedSkeletonHierarchyBlob` as
    this behavior is now intrinsic to the QVVS transform system

## [0.6.6] – 2023-3-13

Officially supports Entities [1.0.0 prerelease 15]

### Fixed

-   Fixed AclUnity compression of scalar clips not initializing safety for the
    compressed clip result
-   Fixed exported bones not being assigned correctly during baking when there
    were more than one in a skeleton

## [0.6.5] – 2023-2-18

Officially supports Entities [1.0.0 prerelease 15]

### Fixed

-   Fixed AclUnity plugins not being configured correctly for desktop linux
    builds
-   Added missing `[DisableAutoCreation]` to `KinemationSmartBlobberBakingGroup`

## [0.6.4] – 2022-12-29

Officially supports Entities [1.0.0 prerelease 15]

### Fixed

-   Fixed children of exported bone Game Objects being reparented directly to
    the skeleton during baking rather than the exported bones
-   Fixed an issue where Kinemation `SuperSystems` may have been sorted by
    attribute when such sorting was not desired

### Improved

-   If for some reason an exported bone is not assigned an index or assigned an
    index to a root, it is now no longer treated as an exported bone and baking
    will restore default transform components for it

## [0.6.2] – 2022-12-10

Officially supports Entities [1.0.0 prerelease 15]

### Improved

-   Kinemation renderer now uses a fixed frame count when recycling rendering
    buffers instead of an async readback, which prevented devices from going
    into a sleep state

## [0.6.1] – 2022-11-28

Officially supports Entities [1.0.0 prerelease 15]

### Added

-   Added `OverrideMeshRendererBase` which if present on a `GameObject` will
    disable default `MeshRenderer` baking for that `GameObject`
-   Added `OverrideMeshRendererBakerBase<T>` which allows baking a `Renderer`
    using a custom mesh and materials that might be generated at bake time

### Fixed

-   Fixed issue where the runtime mechanism for ensuring Kinemation culling
    components were present queried `RenderMesh` instead of `MaterialMeshInfo`
-   Fixed issue where `SkinnedMeshRenderers` might be excluded from
    `RenderMeshArray` post-processing during baking

### Improved

-   Kinemation renderer was updated with the latest material error handling
    improvements of Entities Graphics

## [0.6.0] – 2022-11-16

Officially supports Entities [1.0.0 experimental]

### Added

-   *New Feature:* The Kinemation renderer can now be used in the Editor World,
    providing previews of skinned mesh entities
-   Added `ChunkPerCameraCullingSplitsMask` for shadow map culling
-   Added `CullingSplitElement` for shadow map culling
-   Added many new fields to `CullingContext` which come from the new
    `BatchRendererGroup` API
-   Added `ChunkPerCameraSkeletonCullingSplitsMask` for shadow map culling of
    skeletons

### Changed

-   **Breaking:** Authoring has been completely rewritten to use baking workflow
-   **Breaking:** `KinemationConversionBootstrap.InstallKinemationConversion()`
    has been replaced with
    `KinemationBakingBootstrap.InstallKinemationBakersAndSystems()`
-   **Breaking:** `AnimationClip.ExtractKienamtionClipEvents()` now returns a
    `NativeArray` and expects an `Allocator` argument
-   **Breaking:** Renamed `PhysicsComponentDataFromEntity` to
    `PhysicsComponentLookup` and `PhysicsBufferFromEntity` to
    `PhysicsBufferLookup`
-   Exposed skeleton baking will not include a descendent Game Object with an
    Animator as part of the skeleton
-   Optimized skeleton baking only considers the first level children without
    Skinned Mesh Renderers or Animators to be exported bones, meaning exported
    bones can now have dynamic children
-   *Skeleton Authoring* has been replaced with *Skeleton Settings Authoring*
-   The new Culling algorithms work differently, so custom culling code should
    be re-evaluated as there may be opportunities to leverage more of the system

### Fixed

-   Fixed component conflicts between bound skinned meshes and exported bones
    during baking
-   Fixed bug where exposed bone culling arrays were allocated to double the
    amount of memory they actually required

### Improved

-   Added property inspectors in the editor for `MeshBindingPathsBlob`,
    `SkeletonBindingPathsBlob`, and `OptimizedSkeletonHierarchyBlob` which can
    be used to debug binding issues
-   Many systems are now Burst-compiled `ISystems` using the much-improved
    Collection Component manager and consequently run much faster on the main
    thread, especially during culling with improvements as much as 5 times
    better performance
-   Material Property Component Type Handles are fully cached
-   Meshes no longer need to be marked Read/Write enabled

### Removed

-   Removed `BindingMode`, `ImportStatus`, and `BoneTransformData`

## [0.5.8] – 2022-11-10

Officially supports Entities [0.51.1]

### Fixed

-   Fixed a crash where a `MeshSkinningBlob` is accessed after it has been
    unloaded by a subscene
-   Fixed `LocalToParent` being uninitialized if it was not already present on a
    skinned mesh entity prior to binding
-   Fixed bone bounds not being rebound correctly when a skinned mesh is removed
-   Fixed several caching issues that resulted in bindings reading the wrong
    cached binding data after all entities referencing a cached binding are
    destroyed and new bindings are later created

## [0.5.7] – 2022-8-28

Officially supports Entities [0.51.1]

### Changed

-   Explicitly applied Script Updater changes for hashmap renaming

## [0.5.6] – 2022-8-21

Officially supports Entities [0.50.1] – [0.51.1]

### Added

-   Added `ClipEvents.TryGetEventsRange` which can find all events between two
    time points
-   Added `SkeletonBindingPathsBlob.StartsWith()` and`
    SkeletonBindingPathsBlob.TryGetFirstPathIndexThatStartWith()` for finding
    optimized bones by name.

### Fixed

-   Fixed multiple issues that caused errors when destroying exposed skeleton
    entities at runtime
-   Fixed Editor Mode Game Object Conversion when destroying shadow hierarchies
    used for capturing skeleton structures and animation clip samples

### Improved

-   Improved performance of the culling callbacks by \~15%

## [0.5.5] – 2022-8-14

Officially supports Entities [0.50.1] – [0.51.1]

### Added

-   Added `ClipEvents` which can be baked into `SkeletonClipSetBlob` and
    `ParameterClipSetBlob` instances. These are purely for user use and do not
    affect the Kinemation runtime.
-   Added `UnityEngine.AnimationClip.ExtractKinemationClipEvents()` which
    converts `AnimationEvent`s into a form which can be baked by the Smart
    Blobbers
-   Added `ParameterClipSetBlob` and an associated Smart Blobber. They can be
    used for baking general purpose animation curves and other scalar parameters
    into Burst-friendly compressed forms.
-   Added `BufferPoseBlender.ComputeBoneToRoot()` which can compute a
    `BoneToRoot` matrix for a single bone while the buffer remains in local
    space. This may be useful for IK solvers.
-   Added `SkeletonClipCompressionSettings.copyFirstKeyAtEnd` which can be used
    to fix looping animations which do not match start and end poses

### Fixed

-   Fixed `OptimizedBoneToRoot` construction using `ParentScaleInverse`, which
    was applying inverse scale to translation
-   Fixed the `duration` of clips being a sample longer than they actually are

## [0.5.4] – 2022-7-28

Officially supports Entities [0.50.1]

### Added

-   Added Mac OS support (including Apple Silicon)

### Fixed

-   Fixed an issue where multiple materials on a skinned mesh would not copy
    their skinning indices to the GPU, causing confusing rendering artifacts
-   Fixed crash with Intel GPUs that do not support large compute thread groups

## [0.5.3] – 2022-7-4

Officially supports Entities [0.50.1]

### Fixed

-   Fixed `ShaderEffectRadialBounds` crash caused by broken merge prior to 0.5.2
    release

## [0.5.2] – 2022-7-3

Officially supports Entities [0.50.1]

### Added

-   Added Linux support
-   Added `SkeletonClip.SamplePose()` overload which uses a `BufferPoseBlender`
    and performs optimal pose sampling
-   Added `BufferPoseBlender` which can temporarily repurpose a
    `DynamicBuffer<OptimizedBoneToRoot>` into a working buffer of
    `BoneTransform`s for additive pose sampling (blending) and IK and then
    convert the resulting `BoneTransform`s back into valid `OptimizedBoneToRoot`
    values

### Fixed

-   Fixed `ShaderEffectRadialBounds` which was ignored
-   Fixed issue where root motion animation was applied directly to the
    `OptimizedBoneToRoot` buffer when using pose sampling

### Improved

-   Fixed change filtering of exposed bone bounds
-   ACL Unity plugin binaries are now built using GitHub Actions and have a
    tagged release which includes artifacts for debug symbols as well as
    human-readable assembly text

## [0.5.1] – 2022-6-19

Officially supports Entities [0.50.1]

### Added

-   Added `SkeletonClip.SamplePose()` which uses a fast path to compute the
    entire `OptimizedBoneToRoot` buffer
-   Added bool `hasAnyParentScaleInverseBone` and bitmask array
    `hasChildWithParentScaleInverseBitmask` to `OptimizedSkeletonHierarchyBlob`
    which can be used to detect if an inverse scale value needs to be calculated
    in advance
-   Added `SkeletonClip.boneCount` for safety purposes
-   Added `SkeletonClip.sampleRate` for convenience when using other
    `KeyframeInterpolationMode`s than the default
-   Added `SkeletonClip.sizeUncompressed` and `SkeletonClip.sizeCompressed` to
    compare the efficacies of different compression levels

### Fixed

-   Fixed a bug where only the first skeleton archetype was used for rendering
-   Skinned Mesh Renderers are no longer treated as bones in the skeleton during
    conversion
-   Fixed a bug where the first clip in a `SkeletonClipSetBakeData` was
    duplicated for all of the clips in the resulting `SkeletonClipSetBlob`
-   Scene root transforms are no longer baked into `SkeletonClip`
-   `InstallKinemationConversion()` now works correctly when the conversion
    bootstrap returns `false`
-   Applied the Hybrid Renderer 0.51 specular reflections bugfix to Kinemation’s
    renderer

## [0.5.0] – 2022-6-13

Officially supports Entities [0.50.1]

### This is the first release of *Kinemation*.
