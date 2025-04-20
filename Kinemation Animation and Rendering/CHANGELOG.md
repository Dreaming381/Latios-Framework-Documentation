# Changelog

All notable changes to this package will be documented in this file.

The format is based on [Keep a Changelog](http://keepachangelog.com/en/1.0.0/)
and this project adheres to [Semantic
Versioning](http://semver.org/spec/v2.0.0.html).

## [0.12.6] – 2025-4-20

Officially supports Entities [1.3.14]

### Added

-   Added new Socket workflow for specifying a socket by name

### Fixed

-   Fixed exceptions when adding or regenerating an Optimized Skeleton Cache
-   Reverted a change in 0.12.4 that was intended to work around a Unity bug
    with incorrectly generated skeletons, as this change resulted in the
    generation of unnecessary bones in far more common use cases
-   Fixed authored sockets being treated as imported sockets
-   Fixed a pointer misuse regression in `BufferPoseBlender.Optimize()` which
    caused incorrect behavior for half the bones while stomping on non-bone
    memory

## [0.12.4] – 2025-4-5

Officially supports Entities [1.3.9]

### Added

-   Added `ClipEvents.GetEventIndicesInRange()` which can be used in a `foreach`
    expression

### Fixed

-   Fixed animation clips sometimes dropping the final sample when baking due to
    floating point errors
-   Fixed `RootMotionTools` position offsets being calculated in the wrong
    coordinate space, especially when the root motion contains changing
    directions
-   Worked around a Unity bug where if a skinned mesh and an optimized bone are
    named the same, the skeleton would not be generated correctly

## [0.12.3] – 2025-3-29

Officially supports Entities [1.3.9]

### Added

-   Added `OptimizedSkeletonAspect.IsFinishedWithInertialBlend()`

### Fixed

-   Fixed an issue during skinning where if a skeleton has multiple skinned
    meshes that require both current and previous transforms, the transform
    types would get mixed up

## [0.12.2] – 2025-3-22

Officially supports Entities [1.3.9]

### Fixed

-   Fixed baking skeletons where one of the authoring entities has the
    `BakingOnlyEntityAuthoring` component
-   Fixed adding and removing `EntitiesGraphicsChunkInfo` to all prefabs and
    disabled entities every frame

### Improved

-   Structural change operations inside
    `LatiosUpdateEntitiesGraphicsChunkStructureSystem` are now Burst-compiled
-   Improved a potential single-threaded job bottleneck in scenes with a large
    number of LOD Crossfade entities by splitting the job into two and
    parallelizing one of them

## [0.12.1] – 2025-3-8

Officially supports Entities [1.3.9]

### Added

-   Added scripting define `LATIOS_DISABLE_ACL` which allows the Latios
    Framework to be built for platforms which don’t have a valid ACL plugin as
    long as ACL-backed animation is not used by the project

### Fixed

-   Fixed use of `stackalloc` in a loop in `Ewbik.Solve()` which could result in
    stack out-of-memory errors with high solver iteration counts

## [0.12.0] – 2025-2-23

Officially supports Entities [1.3.9]

### Added

-   *New Feature:* Added nondestructive socket workflow via Socket authoring
    component which can be added to any child Game Object of the optimized
    Animator
-   *New Feature:* Added `SkeletonBoneMaskSetBlob` and smart blobber to bake
    bone masks from Avatar Masks
-   *New Feature:* Added Optimized Skeleton Cache which provides a cached
    version of the optimized skeleton for bakers
-   *New Feature:* Added Unique Mesh components and systems which allow a user
    to write mesh data to dynamic buffers and Kinemation will automatically
    handle pooled mesh creation, registration, and upload
-   *New Feature:* LOD Pack is the new version of what was LOD1 Append and now
    supports up to 3 LOD levels and allows the last level to be an empty fade
    out
-   *New Feature:* Added optional Custom Graphics loop which allows for
    performing a batch round-robin deformation and GPU upload pass on
    user-specified entities before culling for the purposes of custom graphics
-   *New Feature:* Added EWBIK which is a modular IK algorithm for solving both
    position and orientation targets
-   Added `IBaker` extension method `CreateBoneNamesForExposedSkeleton()` which
    allows a baker to get all the bone names and parent indices of an exposed
    skeleton in skeleton order
-   Added `NativeText` and `UnsafeText` extension methods
    `AppendBoneReversePath()` which appends the full reversed path of a bone
    given an array of `SkeletonBoneNameInHierarchy`
-   Added `BufferLookup` version of `BoneNamesRequestHandle.Resolve()` for use
    in baking systems
-   Added `InertialBlendingTransformState.NeedsBlend()` which can be used to
    detect when a blend has finished
-   Added `OverrideMeshInRangeTag` which allows a `MaterialMeshInfo` set up to
    use ranges to replace each mesh in the range with a runtime-registered mesh
    while still using the ranges for materials and submeshes
-   Added `EllipticalSwingTwistConstraint` solver and associated EWBIK
    constraint solver which can model hinge and ball-n-socket joints with a very
    simplistic and intuitive input representation
-   Added skeleton support for `RenderVisibilityFeedbackFlag` which when added
    to a skeleton entity will report whether the skeleton passed culling or was
    used by a rendered skinned mesh
-   Added LATIOS_BL_FORK scripting define which allows Kinemation to work in a
    fork of entities where automatic system job completion of a previous update
    has removed
-   Added `RenderBounds` extension methods `Set()`, `SetMinMax()`, and
    `SetCenterExtents()` which take a Psyshock `Aabb`, a `min` and `max`, and a
    `center` and `extents` respectively
-   Added `GraphicsUnmanaged` methods `CreateMeshes()`, `DestroyMeshes()`,
    `RegisterMeshes()`, and `ApplyAndDisposeWritableMeshData()` which all
    operate on `UnityObjectRef<Mesh>` instances
-   Added `SkeletonClipSetSampler.boneCount` property
-   Added `SkeletonClipSetSampler.Dispose()` overload which allows forcing
    deferred destroying since there are still some situations where Unity
    complains about it

### Changed

-   **Breaking:** Everything once named “Exported Bone” or
    `CopyLocalToParentFromBone` has now been renamed to “Socket” to reflect
    verbiage used at Unite 2024 and align with the rest of the industry
-   **Breaking:** `OverrideMeshRendererBase` and `OverrideHierarchyRendererBase`
    have been renamed to `IOverrideMeshRenderer` and
    `IOverrideHierarchyRenderer` and are now interfaces instead of base classes
-   **Breaking:** Renamed various culling and dispatch super systems, most
    notably the round robin early and late extensions super systems

### Fixed

-   AclUnity has been updated to fixed masked pose sampling being inverted
-   Skinned meshes bound to skeletons now bake with `CopyParentTransformTag`
    when using QVVS Transforms
-   Fixed aliased `AtomicSafetyHandle` secondary version error when baking
    skeleton binding paths, now that the engine checks safety in this scenario
-   Fixed `SkeletonClipSetBlob` smart blobber validation when a clip is `null`
-   Fixed issues where if there are more materials than submeshes provided
    during baking of a renderer, by default the submesh indices assigned to each
    material will be clamped to match Unity Engine behavior, and a new
    `clampSubmeshes` parameter has been added to
    `RenderingBakingTools.ExtractMeshMaterialSubmeshes()` to override this
-   Fixed various bugs with LOD Group baking that would result in large meshes
    up close to the camera being culled
-   Fixed various baking issues with LOD Pack (formerly LOD1 Append)
-   `SkeletonSkinBindingsAspect` was incorrectly requesting write access to the
    internal buffer, preventing it from being used by multiple jobs
    simultaneously
-   Fixed LOD Group culling when there were more than 64 LOD entities in a chunk
-   Fixed wrong `JobHandle` being passed to `BatchRendererGroup` resulting in
    rendering artifacts as apparently Unity does not check this
-   Fixed LOD Pack (formerly LOD1 Append) evaluating culled entities
-   Fixed exception for blend shape meshes computing bounds before their
    rotation buffer has been initialized
-   Fixed skinned meshes having wrong transforms after incremental baking
    because Unity stomps the transform values
-   Fixed exceptions when disabling a rendered entity via the `Disabled`
    component after it has already been rendered once
-   Fixed memory leak when pausing play mode and navigating the scene view by
    using `RateGroupAllocators` for the culling and dispatch system groups
-   Fixed Batch ID management when rapidly spawning and despawning meshes

### Improved

-   In the event that Unity fails to provide a valid blend shape buffer to
    Kinemation during baking, Kinemation now catches the error, logs an error,
    and continues to bake the deforming mesh without blend shapes
-   Changed Kinemation’s authoring component menu items to be more obvious they
    come from Kinemation
-   Runtime entities and entities in closed subscenes now respect F and Shift +
    F hotkeys in the scene view when picked while the scene view is using
    Runtime Data

### Removed

-   Removed 2022 mode of culling and dispatch where dispatch systems updated
    every culling pass

## [0.11.5] – 2024-11-2

Officially supports Entities [1.3.5]

### Fixed

-   Fixed `KinemationPreTransformsBakingGroup` not actually updating before
    transforms, resulting in exposed bones not having QVVS motion history
    components in closed subscenes
-   Fixed GPU corruption which could lead to the editor crashing when an exposed
    bone is missing motion history components in QVVS mode

### Improved

-   Added a check to prevent infinite allocations if more than 1024 culling
    passes are requested since the last `LatiosEntitiesGraphicsSystem` update

## [0.11.4] – 2024-10-19

Officially supports Entities [1.3.5]

### Added

-   Added support for `PerVertexMotionVectors_Tag` introduced in Entities
    Graphics 1.4.2
-   Added `PruneUploadBufferPoolOnFrameCleanup()` to `LatiosSparseUploader`
-   Added `PruneUploadBufferPool()` to `UploadMaterialPropertiesSystem`, which
    provides the `EntitiesGraphicsSystem` equivalent
-   Added name property to `GraphicsBufferUnmanaged`

### Fixed

-   Fixed editor system errors when disabling a `MeshRenderer` that resulted
    from leftover chunk components in incremental bakes

### Improved

-   Added a warning during baking if a material’s shader is `null`

## [0.11.3] – 2024-10-13

Officially supports Entities [1.3.2]

### Fixed

-   Fixed compile errors in
    `ForceInitializeUninitializedOptimizedSkeletonsSystem` when using Unity
    Transforms

### Improved

-   Runtime skinned meshes targeting a prefab or disabled skeleton will now
    result in a failed binding with an explicit error message, rather than
    throwing an obscure exception or crashing

## [0.11.2] – 2024-10-13

Officially supports Entities [1.3.2]

### Fixed

-   Fixed rigid meshes that were also exported bones being treated as skinned
    meshes when using Unity Transforms

### Improved

-   Fixed an internal collection component being gathered with read-write access
    when it was only ever read, potentially resulting in unnecessary job
    dependencies
-   Added `ForceInitializeUninitializedOptimizedSkeletonsSystem` which updates
    during the transform system update on optimized skeletons which have not yet
    been processed by a sync point binding system, which fixes some common
    issues encountered when spawning optimized skeletons

## [0.11.0] – 2024-9-29

Officially supports Entities [1.3.2]

### Added

-   *New Feature:* Added authoring `enum` `MeshDeformDataFeatures` which allows
    specifying the use cases a `MeshDeformDataBlob` needs to support,
    potentially greatly decreasing build size and runtime memory usage
-   *New Feature:* Added support for packing LOD masks per material-mesh-submesh
    and a LOD select inside `MaterialMeshInfo` so that multiple LODs can be
    embedded in a single entity
-   *New Feature:* Added *LOD1 Append* authoring and `MmiRange2LodSelect`
    component for packing two LOD levels into a single entity with crossfade
    support that will be evaluated automatically
-   *New Feature:* Added `GraphicsUnmanaged,` `GraphicsBufferUnmanaged`, and
    extension methods for `UnityObjectRef<ComputeShader>` which allow various
    managed graphics operations to be invoked from Burst-compiled code
-   *New Feature:* Added `RootMotionDeltaAccumulator` and `RootMotionTools` to
    assist in sampling and blending root motion transforms
-   Added `OverrideHierarchyRendererBase` that allows a baker for a
    `GameObject`’s ancestor to override baking for descendant renderers, such as
    combining all descendants into a single entity
-   Added `RenderingBakingTools.IsOverridden()` to help determine if a Renderer
    has an overriding authoring component that should be respected
-   Added `lodMask` to `MeshMaterialSubmeshSettings` and
    `RenderingBakingTools.ExtractMeshMaterialSubmeshes()`
-   Added `RenderingBakingTools.GetDeformFeaturesFromMaterials()` which can
    determine whether `VertexSkinning` or `Deform` feature flags are required by
    the material based on the exposed shader properties
-   Added `RenderingBakingTools.BakeLodMaskForEntity()` which is now responsible
    for traditional LOD Group baking logic
-   Added `LodCrossfade.ToComplement()` to compute a complementary crossfade
    value for a given crossfade
-   Added static class `LodUtilities` for performing common LOD calculations and
    operations
-   Added `UseMmiRangeLodTag` which tells culling to filter each
    mesh-material-submesh by the LOD level stored in `MaterialMeshInfo`
-   Added `MaterialMeshInfo` extension methods for getting and setting a LOD
    select level
-   Added `hasBindPoses` and `hasDeformBoneWeights` to `MeshSkinningBlob`,
    `hasBlendShapes` to `MeshBlendShapesBlob`, `hasMeshNormalizationData` to
    `MeshNormalizationBlob`, and `hasUndeformedVertices` to `MeshDeformDataBlob`
-   Added `ICullingComputeDispatchSystem` and `CullingComputeDispatchData` for
    assisting in writing unmanaged systems that operate in the round-robin super
    systems
-   Added `GenerateBrgDrawCommandsSystem.optimizeForMainThread` which slightly
    improves main thread performance at the cost of reducing parallelism

### Changed

-   **Breaking:** All renderable entities now receive a
    `ChunkPerDispatchCullingMask` which should be used instead of
    `ChunkPerCameraCullingMask` for systems that update inside
    `CullingRoundRobinEarly/LateExtensionsSuperSystems`
-   **Breaking:** `CullingContext` fields
    `globalSystemVersionOfLatiosEntitiesGraphics` and
    `lastSystemVersionOfLatiosEntitiesGraphics` have been moved to a new
    `DispatchContext` component
-   **Breaking:** `GraphicsBufferBrokerReference` has been replaced by
    `GraphicsBufferBroker`, which is now a struct `IComponentData` and returns
    `GraphicsBufferUnmanaged` instances, making it fully Burst-compatible
-   `SkeletonClip.SampleBone()` now sets `worldIndex` to `math.asint(1f)` to
    assist with manual blending algorithms
-   `MeshDeformDataBlob` can now have bindposes but no skinning weights, and it
    can also have empty normalization data and an empty `undeformedVertices`
    array depending on the `MeshDeformDataFeatures` it was created with
-   Many `SubSystems` that performed compute shader dispatches have been changed
    to Burst-compiled `ISystems`
-   `LatiosSparseUploader` has been reworked to use the new unmanaged graphics
    APIs

### Fixed

-   Fixed various edge cases with `LodCrossfade` values

### Improved

-   Improved culling performance across many systems as well as culling callback
    setup and teardown by Burst-compiling more things
-   Improved shadowmap culling performance significantly in Unity 6 by
    aggregating compute dispatches using the new `OnFinishedCulling(`) callback
-   Skinning now respects skinned entities that were explicitly enabled for
    rendering by user code even if the skeleton did not pass frustum culling
-   Extension method `TransformQvvs.NormalizeBone()` now explicitly ensures the
    weight value is `1f `after normalization

### Removed

-   **Breaking:** Removed `lodSettings` from `MeshRendererBakeSettings` as the
    functionality has been moved to
    `RenderingBakingTools.BakeLodMaskForEntity()` instead
-   **Breaking:** Removed shader subgraph URP14-LOD_Crossfade_Fix_Node as URP
    now supports LOD Crossfade natively
-   **Breaking:** Renamed `LodCrossfade` method `SetHiResOpacity()` to
    `SetFromHiResOpacity()`

## [0.10.7] – 2024-8-25

Officially supports Entities [1.2.1] – [1.2.4]

### Fixed

-   Fixed an issue where the `SceneBoundingVolume` was populated with NaNs when
    only entities centered at the origin were included in the subscene

## [0.10.5] – 2024-7-28

Officially supports Entities [1.2.1] – [1.2.3]

### Fixed

-   Backported lightmap material fix from Entities Graphics 1.3.0-pre.4
-   Fixed skeleton shadows only being drawn when all skeletons in a chunk were
    completely within the shadow culling planes

## [0.10.4] – 2024-7-20

Officially supports Entities [1.2.1] – [1.2.3]

### Fixed

-   Fixed exceptions during baking for null animators, where an error was
    supposed to be logged instead

## [0.10.3] – 2024-6-30

Officially supports Entities [1.2.1] – [1.2.3]

### Added

-   Added overload of `OptimizedSkeletonAspect.StartNewInertialBlend()` which
    accepts a mask to only apply the blend to specific bones

### Fixed

-   Fixed Latios Vertex Skinning node for meshes which used fewer than 3 real
    bone influences per vertex across the entire mesh
-   Fixed an issue where the history of all mesh uploads was uploaded whenever
    new uploads were requested
-   Fixed GPU memory recycling issues where bindposes would often fail to reuse
    memory while bone weights would allocate in too small of regions and clobber
    memory, which would result in GPU crashes

## [0.10.2] – 2024-6-22

Officially supports Entities [1.2.1] – [1.2.3]

### Fixed

-   Fixed SpeedTree LOD Crossfade values not being submitted

## [0.10.1] – 2024-6-9

Officially supports Entities [1.2.1]

### Fixed

-   Fixed rotations not being normalized when multiple bone animation sample
    weights sum up to exactly 1f
-   Fixed Dual Quaternion Skinning node option in shader graph Latios Vertex
    Skinning node generating invalid code

### Improved

-   Eliminated a warning in Rider about OptimizedSkeletonAspect

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
