# Changelog

All notable changes to this package will be documented in this file.

The format is based on [Keep a Changelog](http://keepachangelog.com/en/1.0.0/)
and this project adheres to [Semantic
Versioning](http://semver.org/spec/v2.0.0.html).

## [0.14.10] – 2026-1-17

Officially supports Entities [1.4.4]

### Added

-   Added `ComponentBroker` method `IsAlive()` to match `EntityManager` and
    `EntityStorageInfoLookup` extension methods
-   Added Exposed `EntityManager` extension method `GetComponentDataRW<T>()`
    which is useful for unifying wrapper interface APIs

## [0.14.9] – 2026-1-11

Officially supports Entities [1.4.4]

This release only contains internal compatibility changes

## [0.14.8] – 2026-1-3

Officially supports Entities [1.4.3]

### Added

-   Added `TlsfAllocator` for when you need a fast Collections
    package-compatible allocator into pools of pre-allocated memory that is only
    accessed by a single thread at a time
-   Added `DynamicMultiList<T>` for when you need multiple dynamically-sized
    lists packed into a single `DynamicBuffer`
-   Added `Exposed` extension `EntityStorageInfoLookup.IsCreated()` which is
    useful in union types
-   Added Collections package bug workaround method `RemoveSafetyHandles` in
    `Exposed`, which you should probably never call

## [0.14.6] – 2025-12-20

Officially supports Entities [1.4.3]

### Added

-   *New Feature:* Added `IInstantiateCommand` and new
    `InstantiateCommandBufferCommand1` and `InstantiateCommandBufferCommand2`
    types for allowing custom post-processing of instantiated entities upon
    playback
-   Added `ThreadStackAllocator` property `isCreated` for lazy operations
-   Added `EntityManager` and `EntityStorageInfoLookup` extension method
    `IsAlive()` which can check whether an entity exists and is not pending
    destruction due to cleanup components
-   Added `Exposed` `EntityArchetype` extension `IsCleanup()` to detect
    archetypes representing cleanup entities
-   Added `BootstrapTools` method `IsAssemblyReferencingOtherAssembly()` as a
    strongly-typed alternative to `IsAssemblyReferencingSubstring()`
-   Added `ComponentBroker` methods `TryWriteComponentIgnoreParallelSafety()`
    and `TryWriteBufferIgnoreParallelSafety()` intended to assist
    deserialization operations
-   Added Exposed `EntityManager` extension methods `GetComponentLookup<T>()`
    and `GetEntityStorageInfoLookup()`
-   Added `UnityObjectRef` extension method `EntityID()` in Unity 6.3 and newer

### Removed

-   **Breaking:** Removed exposed `BlobAssetStore` Burst extension methods as
    Entities 1.4.3 allows normal `BlobAssetStore` methods to be Burst-compatible

## [0.14.5] – 2025-12-13

Officially supports Entities [1.3.14]

### Added

-   Added optional parameter bool `readOnly` to
    `LatiosWorldUnmanaged.GetCollectionComponent()`
-   Added `ComponentSystemGroup` tracing APIs to `Unity.Entities.Exposed` for
    Unity 6.4 compatibility

### Fixed

-   Fixed `DestroyCommandBuffer` memory corruption when destroying enough
    entities to trigger job dispatch

### Improved

-   Added additional safety checks inside `DestroyCommandBuffer` to detect
    invalid `LinkedEntityGroup` buffers
-   Improved the performance of `InstantiateCommandBuffer` playback

## [0.14.4] – 2025-11-16

Officially supports Entities [1.3.14]

### Fixed

-   Fixed `DestroyCommandBuffer` job safety error due to use of wrong allocator
    type

## [0.14.3] – 2025-11-15

Officially supports Entities [1.3.14]

### Fixed

-   Fixed disposal memory leak in `UnsafeIndexedBlockList` if an index had no
    more blocks but still had a valid block list

## [0.14.2] – 2025-11-9

Officially supports Entities [1.3.14]

### Fixed

-   Fixed disposal memory leak in `UnsafeIndexedBlockList` if indices were
    cleared on an instance backed by an allocator requiring disposal

## [0.14.1] – 2025-11-2

Officially supports Entities [1.3.14]

### Added

-   Added a new `Playback()` overload to `DestroyCommandBuffer` which employs a
    new typically faster batching algorithm that may sometimes perform
    operations multi-threaded
-   Added a new static method `DestroyEntitiesWithBatching()` method to
    `DestroyCommandBuffer` which performs the new batching algorithm on a custom
    `NativeArray<Entity>`
-   Added `BlobBuilder` extension method `ConstructFromSpan<T>()` as a
    `ReadOnlySpan` alternative to the `NativeArray` counterpart extension method
-   Added `EntityManager Entities.Exposed` extension method
    `GetChunkFromChunkHashcode()`

### Fixed

-   Worked around an issue where Entities 1.4.x detection wasn’t being
    recognized for Entities Exposed by now detecting the missing scripting
    define symbol and displaying an error
-   Fixed C\#10 file scope namespaces not being handled correctly in source
    generators

### Improved

-   `SyncPointPlaybackSystem` now uses the new `DestroyCommandBuffer` batching
    algorithm for improved performance

## [0.14.0] – 2025-10-18

Officially supports Entities [1.3.14]

### Added

-   *New Feature:* Added `UnsafeParallelBlockList<T>` and
    `UnsafeIndexedBlockList<T>` which wrap the untyped versions
-   *New Feature:* Added `AtomicDoubleBuffer<T>` which is a multi-producer
    single-consumer type intended for use inside a `SharedStatic<T>` as part of
    the implementation of a static class utility
-   Added `Sort()` extension methods to `Span<T>` that use Unity’s native
    sorting algorithm
-   Added `UnsafeParallelBlockList.Clear()` to remove all elements
-   Added `LockIndexReentrant()` and `UnlockIndex()` methods to
    `UnsafeParallelBlockList`, which allows for better coordination between
    readers and writers
-   Added LATIOS_BURST_DETERMINISM support

### Changed

-   Changed `UnsafeParallelBlockList.Write<T>()` to now take the value as an
    `in` parameter

### Removed

-   Moved `simdFloat3`, `simd`, `Rng`, `RngToolkit`, `Qcp`, `LatiosMath`, and
    `MathExtensions` to Calci

## [0.13.7] – 2025-9-13

Officially supports Entities [1.3.14]

### Fixed

-   Fixed element removal in `DynamicHashMap` corrupting the hashmap when the
    element is one of three or more elements using the same bucket, and is not
    the last element added

## [0.13.6] – 2025-9-6

Officially supports Entities [1.3.14]

### Added

-   Added various `ArchetypeChunk` extension APIs that accept a
    `ComponentBroker` for accessing component array pointers, shared components,
    and `EnabledMasks`

### Changed

-   `ComponentBroker` APIs that access types in read-only mode will never bump
    the change version, even if the type was registered with the
    `ComponentBroker` using read-write access, though performance may be worse
    in such a scenario

### Fixed

-   Fixed consecutive additions of unique values into `DynamicHashMap` where the
    unique keys of these additions all collide in the same bucket would result
    in some of these elements no longer being retrievable by key

## [0.13.3] – 2025-8-16

Officially supports Entities [1.3.14]

### Added

-   Added `IBaker` extension methods `ShouldBakeAll<T>()` and
    `ShouldBakeAllInterface<T>()` which are designed to help users bake multiple
    instances of a component on a `GameObject` at once, allowing for example all
    the results to be packed into a single `DynamicBuffer`
-   Added `GetUnsafeComponentPtrXXX()` methods to `ComponentBroker`, which allow
    for accessing pointers to components type-agnostically

### Fixed

-   Fixed `IBaker` extension methods that operated on interfaces registering the
    wrong dependencies when requested to access parent or children components

## [0.13.1] – 2025-8-2

Officially supports Entities [1.3.14]

### Fixed

-   Fixed Burst logging of entity errors for some collection component
    interactions

## [0.13.0] – 2025-7-6

Officially supports Entities [1.3.14]

### Added

-   *New Feature:* Added `GapAllocator` for allocating into spans of specialized
    resources such as graphics buffers with minimal fragmentation
-   Added `CountForThreadIndex()` method to `UnsafeParallelBlockList`
-   Added `BlackboardEntity` constructor which accepts a `SystemHandle`,
    allowing a blackboard entity API for the system entity
-   Added `BlackboardEntity` overloads for `AddComponent()` and
    `RemoveComponent()` which accept a `ComponentTypeSet`
-   Added `BlackboardEntity` methods `GetEnabled<T>()` and `SetEnabled<T>()` for
    working with `IEnableableComponent`
-   Added methods `GetOrAdd()` and `TryGet()` to `DynamicHashMap` which allow
    for more complex operations on `DynamicHashMap`s with fewer lookups
-   Added `simd` methods `distance()` and `clamp()`
-   Added `UnsafeList<T>` extension method `AddRangeFromBlob()`
-   Added `Exposed` APIs for working with `BlobAssetOwner` components
-   Added `GetEntity()` extension method to `SystemHandle` to retrieve the
    underlying entity

### Fixed

-   Fixed `Bits.SetBits()` overload which accepts a `ulong`
-   Fixed compilation issues in ShaderLibrary hlsl files

### Improved

-   `AddOrSetCollectionComponentAndDisposeOld<T>()` will now only create a sync
    point if the entity does not already have both the Exists and Cleanup
    components, and both will be added in a single structural change to reduce
    the total archetypes

## [0.12.6] – 2025-4-20

Officially supports Entities [1.3.14]

### Added

-   Added shorthand `GetCollectionAspect<T>()` method to `BlackboardEntity`
-   Added `FluentQuery.WithAspectPresent<T>()` which adds all required `IAspect`
    component types but ignores the enabled state of those components
-   Added Color and Color32 extension methods `ToFloat4()` and `ToHalf4()`

### Removed

-   Removed `FixBakingAllocatorSystem` as this system’s sole purpose was to
    address an issue that has now been fixed in the new Entities package

## [0.12.5] – 2025-4-12

Officially supports Entities [1.3.9]

### Added

-   Added `AddComponentsCommandBuffer` which does not perform any component
    initialization, but still respects the
    `AddComponentsDestroyedEntityResolution` it is constructed with
-   Added `DynamicHashMap.ReconstructAfterRemap()` which can be used to rehash
    all elements due to hashcodes changing after an entity remap or blob asset
    deserialization

## [0.12.3] – 2025-3-29

Officially supports Entities [1.3.9]

### Added

-   Added `ThreadStackAllocator.AllocateAsSpan<T>()` as a convenient alternative
    to receiving a pointer directly
-   Added Edit Menu toggle *Disable Subscene Reimport During Playmode* which
    prevents closed subscene reloads when editing a prefab while in play mode

### Improved

-   Improved the error message when the framework encounters a bootstrap
    ambiguity due to multiple bootstraps being present in a project

## [0.12.2] – 2025-3-22

Officially supports Entities [1.3.9]

### Added

-   Added `entityStorageInfoLookup` property to `TempQuery`
-   Added `SetArchetypes()` to `TempQuery` which allows replacing the archetype
    array used by the `TempQuery` without having to reconstruct the internal
    component type maps

## [0.12.1] – 2025-3-8

Officially supports Entities [1.3.9]

### Fixed

-   Fixed collection components sometimes not being default-initialized when
    created via adding an `ExistComponent` in a baker or `EntityCommandBuffer`

## [0.12.0] – 2025-2-23

Officially supports Entities [1.3.9]

### Added

-   *New Feature:* Added `IBaker` extension methods for getting components by
    interface with correct incremental baking dependency registration
-   *New Feature:* Added `Bits` for packing bitfields and small `enum`s into
    various integer data types
-   *New Feature:* Added `ComponentBroker` for sending a large amount of
    arbitrary component types into a job which can be accessed generically
-   *New Feature:* Added `TempQuery` and associated enumerators which allow
    iterating entities from a temporary entity query while in a job
-   *New Feature:* Added ShaderLibrary hlsl files for working with `quaternion`
    and `RigidTransform` types featured in the Unity Mathematics package
-   Added `ParallelDetector` which is a safety checks container that tries to
    detect if the job is a parallel job, though it has some caveats for
    `IJobFor`
-   Added `GetCurrentPtr()` to `UnsafeIndexedBlockList` enumerator types
-   Added `CoreBootstrap.InstallNetCodePreSpawnEnableInEditorSystem()` which
    enables all pre-spawned ghosts in the editor world
-   Added `LatiosWorldUnmanaged.CompleteAllTrackedJobs()` to complete all jobs
    registered with collection components
-   Added `TypePack` which can be constructed with less boilerplate than
    `ComponentTypeSet` and implicit cast into one or a
    `FixedList128Bytes<ComponentType>`
-   Added extension methods to `EntityManager`, `EntityCommandBuffer`, and
    `BlackboardEntity` for adding multiple components specified as generic
    arguments
-   Added `PostSyncPointGroup` for scheduling jobs after the sync point but
    before simulation or rendering
-   Added `SystemState` extension `GetLiveBakeSafeLastSystemVersion()` because
    incremental baking will silently clobber data without triggering change
    filters
-   Added `IBaker` extension method `GetAuthoringGameObjectWithoutDependency()`
-   Added `ArchetypeChunk` extension method `GetBufferAccessor<T>()` which
    accepts a `DynamicComponentTypeHandle`
-   Added various extension methods for `EntityArchetype` to extract interesting
    metadata about the archetype
-   Added `DynamicHashMap` attributes to help make elements more
    inspector-friendly for debugging
-   Added `EntityManager` extension method `CompleteDependencyBeforeRO()` which
    accepts a `TypeIndex` parameter

### Changed

-   **Breaking:** `ICustomEditorBootstrap` has been reworked such that it is now
    responsible for creating the original editor world
-   Havok systems are now injected in `BootstrapTools.InjectUnitySystems()`
-   Moved the Restart Editor World edit menu option under a Latios category
-   Specified in the documentation that reactive systems in
    `LatiosWorldSyncGroup` should now use `OrderLast = true` while systems
    responsible for spawning and despawning should not
-   Removed authoring namespace from extension method
    `UnityEngine.Object.DestroySafelyFromAnywhere()`

### Fixed

-   Added a baking system to properly rewind `WorldUpdateAllocator` so that it
    can be used properly during incremental baking without constantly allocating
    new data on rebakes
-   Fixed smart blobber memory leak where multiple identical blobs generated in
    the same baking pass would not all be disposed
-   Fixed issue where the non-Latios default editor world would pick up systems
    using the Latios Framework, causing brief error messages before the editor
    world would be replaced by the one in `ICustomEditorBootstrap`
-   Fixed missing safety check in
    `LatiosWorldUnmanaged.UpdateCollectionComponentMainThreadAccess<T>()`
-   Fixed various `DynamicHashMap` bugs
-   Fixed source generators to compensate for IL2CPP change where code stripping
    began stripping methods from cleanup components

### Improved

-   Changed Core’s authoring component menu items to be more obvious they come
    from Latios Core
-   Source generators will now add missing `[BurstCompile]` attributes to
    containing types of collection components

## [0.11.3] – 2024-10-13

Officially supports Entities [1.3.2]

### Fixed

-   Fixed a compiler warning in `LatiosClientServerBootstrap` for NetCode
    projects

## [0.11.1] – 2024-10-5

Officially supports Entities [1.3.2]

### Fixed

-   Fixed memory stomps in `UnsafeIndexedBlockList` methods
    `ConcatenateAndStealFromUnordered` and `MoveIndexToOtherIndexUnordered`

## [0.11.0] – 2024-9-29

Officially supports Entities [1.3.2]

### Added

-   *New Feature:* Added *NetCode QVVS – Explicit Workflow* bootstrap template
-   *New Feature:* Added `Qcp.Solve()` which can compute a rotation or
    `RigidTransform` that minimizes the root-mean-square distance of a set of
    to-be-transformed points to their corresponding set of target points in a
    pair-wise manner
-   Added `UnsafeIndexedBlockList` methods `LockIndexRenetrant()` and
    `UnlockIndex()` to facilitate `SharedStatic` producer-consumer patterns
-   Added `BootstrapTools.AddGroupAndAllNestedGroupsInsideToFilter<>()` which
    can be used to gather a collection of `ComponentSystemGroup` types
    underneath a key type to easily check if any other system type will be
    injected into a system within the update span of the specified key group
    type
-   Added `LatiosWorld.LoadFromResourcesAndPreserve<>()` for systems that load
    assets from Resources in `OnCreate()` and store them in `UnityObjectRef`, as
    doing this can cause Unity to crash prior to 2022.3.43f1 and 6000.0.16f1
-   Added `LatiosWorldUnmanaged.latiosWorld` property to retrieve the managed
    `LatiosWorld` instance
-   Added `Compatibility.UnityNetCode.InputDeltaUtilities` based on the Unity
    Character Controller NetCode samples
-   Added a new interface `ISpecifyDefaultVariantsBootstrap` which allows for
    registering NetCode variants in the bootstrap rather than a custom system
-   Added `Compatibility.UnityNetCode.NetCodeBootstrapTools` which provides
    various NetCode-specific utilities for bootstrap code

### Changed

-   **Breaking:** By default, a client world will now always override the
    `DefaultGameObjectInjectionWorld`
-   NetCode standard injection bootstrap template has been updated to use phased
    system initialization similar to the other bootstraps

### Fixed

-   Fixed NetCode compatibility with
    `CoreBakingBootstrap.ForceRemoveLinkedEntityGroupOfLength1()` by always
    preserving `LinkedEntityGroup` when `LinkedEntityGroupAuthoring` is present

### Improved

-   Added a small optimization when a system writes a Collection Component and
    finishes with a `default` `JobHandle` for `Dependency` which avoids sending
    superfluous updates to the job system

### Removed

-   Removed NetCode check in `CoreBootstrap.InstallSceneManager()`, as this is
    now experimentally supported

## [0.10.7] – 2024-8-25

Officially supports Entities [1.2.1] – [1.2.4]

### Fixed

-   Fixed optimization systems injected via custom baking bootstraps not being
    injected in the `OptimizationGroup`

## [0.10.6] – 2024-8-3

Officially supports Entities [1.2.1] – [1.2.3]

### Fixed

-   Fixed several missing internal state updates inside `DynamicHashMap`

## [0.10.4] – 2024-7-20

Officially supports Entities [1.2.1] – [1.2.3]

### Added

-   `EntityWith<>` and `EntityWithBuffer<>` now implement various `IEquatable<>`
    and `IComparable<>` interfaces

## [0.10.2] – 2024-6-22

Officially supports Entities [1.2.1] – [1.2.3]

### Added

-   Added `CoreBakingBootstrap.ForceRemoveLinkedEntityGroupsOfLength1() `which
    can significantly reduce heap allocations at runtime

## [0.10.1] – 2024-6-9

Officially supports Entities [1.2.1]

### Fixed

-   Fixed broken `DynamicHashMap` resize logic

### Improved

-   `RngToolkit` `AsFloat()` and related methods no longer require
    `minInclusive` to be less than `maxExclusive`, matching the behavior of
    `Unity.Mathematics.Random`

## [0.10.0] – 2024-5-26

Officially supports Entities [1.2.1]

### Added

-   *New Feature:* Added `UnsafeIndexedBlockList` which allows for a
    user-specified number of blocklists to be stored within the container with
    support for direct aligned reading and writing
-   *New Features:* Added `AddComponentsCommndBuffer` which supports
    initializing values and has special options for when the entity is destroyed
    prior to playback to aid with cleanup component preservation
-   Added Subscene Load Options for authoring and
    `DisableSynchronousSubsceneLoadingTag` for runtime to allow some
    automatically loaded subscenes to be loaded asynchronously when using Scene
    Management
-   Added `ThreadStackAllocator` which provides a per-thread scoped memory
    allocation mechanism for advanced use cases
-   Added `SystemState` extension methods `InitSystemRng()`, `GetJobRng()`, and
    `GetMainThreadRng()` to simplify `SystemRng` usage
-   Added `float2` extension method `x0y()` to convert from 2D to xz-planar 3D
-   Added `BlobArray` extension method `AsSpan()`
-   Added `NativeList` extension method `Clone()`
-   Added `CollectionsExtension.CombineDependencies()` method which can accept
    `Span<JobHandle>`
-   Added `EntityQuery` extension method `UsesEnabledFiltering()` which allows
    for quickly determining if the query has any components that expect a
    particular enabled state
-   Added `UnityObjectRef` extension method `InstanceID()` which allows
    obtaining the instance ID of the stored object

### Changed

-   **Breaking:** The scripting define `ENTITY_STORE_V1` is now required when
    the Latios Framework is installed
-   **Breaking:** Renamed `UnityEngine.Object` extension method
    `DestroyDuringConversion()` to `DestroySafelyFromAnywhere()`
-   **Breaking:** `UnsafeParallelBlockList` is now a wrapper around
    `UnsafeIndexedBlockList` and uses the `UnsafeIndexedBlockList` Enumerator
    and `ElementPtr` types in its API

### Fixed

-   Fixed an internal flagging issue with `DynamicHashMap` resulting in
    incorrect behavior
-   Fixed wrong namespace for `AutoDestroyExpirablesSystem`

### Improved

-   *New Feature:* All `ComponentSystemGroup` types can properly update systems
    making use of Latios Framework features such as conditional updates and
    dependency tracking
-   Collection Components codegen no longer requires the assembly be compiled
    with `unsafe` support
-   Collection Components codegen should now support higher IL2CPP code
    stripping levels

## [0.9.1] – 2024-1-27

Officially supports Entities [1.1.0-pre.3]

### Fixed

-   Fixed finding of NetCode bootstraps in builds

## [0.9.0] – 2024-1-20

Officially supports Entities [1.1.0-pre.3]

### Added

-   *New Feature:* Added `IAutoDestroyExpirable` interface which inherits
    `IEnableableComponent` and allows an entity to be automatically destroyed
    when all deriving components are disabled
-   *New Feature:* Added `ICollectionAspect` interface which allows modules to
    expose abstractions of otherwise internal collection components and can be
    accessed from `LatiosWorldUnmanaged`
-   *New Feature:* Re-added NetCode support via `LatiosClientServerBootstrap` as
    well as additional interfaces `ICustomLocalWorldBootstrap`,
    `ICustomClientWorldBootstrap`, `ICustomServerWorldBootstrap`, and
    `ICustomThinClientWorldBootstrap`
-   Added `IBaker.RequestCreateBlobAsset()` extension that outputs the blob
    baking entity to simplify scaffolding of user-made smart blobbers
-   Added `BootstrapTools.InjectUserSystems()` which injects systems not
    installed by `InjectUnitySystems()`, and allows for user systems to be
    injected after module installers
-   Added `ThinClient` as `WorldRole` option when creating a `LatiosWorld`
-   Added `SuperSystem.DoLatiosFrameworkComponentSystemGroupUpdate()` for
    updating all systems in a `ComponentSystemGroup` while supporting Latios
    Framework features
-   Added `simd` methods `normalize()`, `normalizesafe()`, and `csumabcd()`
-   Added support for copying enabled states for components and buffers in
    `EntityManager` extension methods, `EntityDataCopyKit`, and when merging
    blackboard entities
-   Added `IBaker` extension `GetAuthoringObjectForDebugDiagnostics()` which
    does not define any dependency

### Changed

-   **Breaking:** Fluent Queries have been completely redesigned to better
    account for enabled states and aspects
-   **Breaking:** Changed `ArchetypeChunk` method `GetChunkPtrAsUlong()` to
    `GetChunkIndexAsUint()`
-   **Breaking:** Replaced `SystemSortingTracker` with a new bool property
    `isSystemListSortDirty` on `ComponentSystemGroup`
-   The bootstrap templates have been redesigned to better serve Unity
    Transforms and Injection Workflow use cases

### Fixed

-   Added missing safety check to
    `LatiosWorldUnmanaged.UpdateCollectionComponentDependency()`

### Improved

-   *New Feature:* All `ComponentSystemGroup` types can properly update systems
    making use of Latios Framework features such as conditional updates and
    dependency tracking

### Removed

-   Removed defunct `LatiosClientServerBootstrapBase` and base NetCode system
    types
-   Removed `IRateManager` instances on `SimulationSystemGroup` and
    `PresentationSystemGroup`

## [0.8.0] – 2023-9-23

Officially supports Entities [1.0.16]

### Added

-   Added `ISmartPostProcessItem` and `IBaker` extension `AddPostProcessItem()`
    where each item will receive a callback after Smart Blobbers have been
    processed, similar to the second step of `ISmartBakeItem`
-   Added `UnsafeParallelBlockList.CopyElementsRaw()` which can copy all
    elements into a contiguous memory region
-   Added `DynamicHashMap`, a hash map wrapper around a `DynamicBuffer` that
    serializes properly with Entity and blob asset references
-   Added `ShuffleElements()` to all `Rng` flavors for reordering a list of
    elements randomly

### Changed

-   The `sceneBlackboardEntity` creation now happens at a fixed point in the
    frame right before blackboard entity merging
-   The `OnNewScene()` callbacks in `SubSystem` and `ISystemNewScene` now happen
    at a fixed point in the frame right after blackboard entity merging

### Fixed

-   Added `[MonoPInvokeCallback]` attributes to source generated Burst function
    pointer methods for IL2CPP builds

### Improved

-   Updated parameter names of `Rng` methods to specify inclusivity and
    exclusivity

### Removed

-   Removed `SmartBaker` virtual method `RunPostProcessInBurst()`

## [0.7.7] – 2023-8-15

Officially supports Entities [1.0.14]

### Added

-   Added `isCreated` property to `UnsafeParallelBlockList`
-   Added `GetSharedComponentDataManaged()` and
    `SetSharedComponentDataManaged()` to `BlackboardEntity`
-   Added support for merging managed components onto blackboard entities

### Fixed

-   Worked around a Unity regression with `TypeManager.GetAllSystems()` that
    caused extra interface methods to not work correctly when the system was
    decorated with `[DisableAutoCreation]`

### Improved

-   Latios Framework source generators now only run on assemblies referencing
    the Latios Framework

## [0.7.6] – 2023-7-9

Officially supports Entities [1.0.11]

### Fixed

-   Fixed a memory leak in `SyncPointPlaybackSystem`

## [0.7.4] – 2023-6-18

Officially supports Entities [1.0.10]

### Fixed

-   Fixed a missing `[BurstCompile]` attribute in `DestroyCommandBuffer`

## [0.7.3] – 2023-6-10

Officially supports Entities [1.0.10]

### Fixed

-   Fixed several issues regarding memory management when playing back custom
    command buffers without Burst by switching from jobs to Burst-compiled
    methods
-   Fixed a missing folder warning by deleting an empty folder in
    EntitiesExposed

## [0.7.2] – 2023-6-4

Officially supports Entities [1.0.10]

### Fixed

-   Fixed bad using statement in BlackboardEntity.cs

## [0.7.1] – 2023-6-3

Officially supports Entities [1.0.10]

### Added

-   Added `LatiosWorldUnmanaged.UpdateCollectionComponentMainThreadAccess() `and
    equivalents for `BlackboardEntity` and `EntityManager` for declaring that
    there were no jobs scheduled for a collection component

### Improved

-   Added referral to documentation for automatic dependency management error
    messages

## [0.7.0] – 2023-5-29

Officially supports Entities [1.0.10]

### Added

-   Added `optimizationSystemTypesToDisable` and`
    optimizationSystemTypesToInject` to `CustomBakingBootstrapContext`
-   Added default `Dispose()` method to `IManagedStructComponent` which can be
    reimplemented for object pooling purposes
-   Added `UnsafeParallelBlockList.AllThreadsEnumerator` to enumerate the entire
    container on a single thread
-   Added `BlackboardEntity.RemoveComponent<T>()` which was missing for some
    reason
-   Added `BootstrapTools.IsAssemblyReferencingSubstring()` which allows
    searching for assemblies referencing assemblies other than Latios Framework
    assemblies
-   Added `LatiosWorldUnmanaged.isValid` to check if a cached
    `LatiosWorldUnmanaged` instance was actually initialized and not left as
    default

### Changed

-   **Breaking:** Most bootstrap APIs now use `SystemTypeHandle` instead of
    `System.Type`
-   **Breaking:** `ICollectionComponent` and `IManagedStructComponent` now rely
    on source generators and must be declared as partial
-   **Breaking:** `ICollectionComponent` and `IManagedStructComponent` no longer
    use `AssociatedComponentType` and instead define a nested `ExistComponent`
    via source generators which can be referenced in user code
-   **Breaking:** `EntityWith` and `EntityWithBuffer` now have special accessor
    methods instead of the indexer and require the lookup to all methods be
    passed by ref, as the previous methods and indexer did not take advantage of
    the lookup caches
-   Updated all templates to be compatible with the new changes
-   `InitializationSystemGroup`, `SimulationSystemGroup`, and
    `PresentationSystemGroup` now have custom `IRateManager` instances applied
    to them

### Fixed

-   Fixed baking override leaks
-   Fixed issue registering a Smart Blobber in `OnCreate()` when the Smart
    Blobber system was injected before the `SmartBlobberCleanupBakingGroup`
-   Fixed leak in `EnableCommandBuffer` playback
-   Fixed crash in `UnsafeParallelBlockList.Enumerator` when a thread index is
    empty
-   Fixed leak in the mechanism used to call `OnNewScene()` and
    `ShouldUpdateSystem()` callbacks on unmanaged systems
-   Fixed leaks in teardown of collection component and managed struct component
    storages
-   Fixed leaks in teardown of `SyncPointPlaybackSystem` if pending buffers
    still are present at shutdown

### Improved

-   Improved XML documentation coverage
-   Fluent queries are now Burst-compatible
-   Instead of a bunch of generic systems inside
    `ManagedComponentsReactiveSystemGroup`, there are now just
    `CollectionsComponentsReactiveSystem` and
    `ManagedStructComponentsReactiveSystem` which rely less on reflection and
    more on Burst

### Removed

-   Removed `BootstrapTools.PopulateTypeManagerForGenerics()` as incremental
    source generators are a more stable solution to the problems this method
    solved
-   Fluent Queries no longer have shared components or change filters as part of
    their API. These should be added to the query after creating it via fluent
    queries.
-   Removed `LatiosInitializationSystemGroup`, `LatiosSimulationSystemGroup`,
    and `LatiosPresentationSystemGroup`, as the base classes are used instead
-   Removed Improved Transforms and Extreme Transforms functionality, as such
    functionality now exists in the QVVS Transforms module

## [0.6.5] – 2023-2-18

Officially supports Entities [1.0.0 prerelease 15]

### Added

-   Added `ArchetypeChunk` exposed extensions `GetMetaEntity()` and`
    GetChunkPtrAsUlong()`
-   Added `float3x4.Scale()`

### Changed

-   Bootstrap Templates no longer require unsafe code enabled

### Fixed

-   Fixed `PreSyncPointGroup` sorting warnings
-   Fixed various code compilation issues when creating builds
-   Fixed exception thrown while destroying a `LatiosWorldUnamanged` if the
    project does not have any managed structs

### Improved

-   System sorting has been disabled for disabled for
    `ManagedComponentsReactiveSystemGroup`, reducing startup performance
-   Removed unnecessary system sorting calls when `enableSorting` is false and
    systems were added but not removed

## [0.6.4] – 2022-12-29

Officially supports Entities [1.0.0 prerelease 15]

### Added

-   Added `FluentQuery.IgnoreEnableableBits()` which adds
    `EntityQueryOptions.IgnoreComponentEnabledState` to the query
-   Added `IBakerExposedExtensions.GetDefaultBakerTypes()` to investigate all
    types of bakers that will be created by default by Unity
-   Added `math.InverseRotateFast()` methods which assume the passed in
    quaternions are normalized and because of this perform faster inverse
    operations

### Changed

-   Renamed `FluentQuery.IncludeDisabled()` to`
    FluentQuery.IncludeDisabledEntities()`

### Fixed

-   Fixed `ICustomBakingBootstrap` ignoring `GameObjectBaker` types and not
    respecting derived type sorting
-   Fixed Smart Blobbers pre-maturely disposing shared blob assets
-   Fixed `SuperSystem` overwriting an override `EnableSystemSorting = true`
    inside of `CreateSystems()`
-   Fixed Improved Transforms warnings about caching type handles

## [0.6.1] – 2022-11-28

Officially supports Entities [1.0.0 prerelease 15]

### Added

-   Added `SystemRng` which provides a new workflow for `Rng` inside of
    `IJobEntity`
-   Added `Rng` constructor overload which accepts a `FixedString128Bytes`
-   Added `float3x4.TRS()` extension method

### Changed

-   `CurrentScene` is now updated before `OnNewScene()` callbacks

### Fixed

-   Fixed script templates which attempted to load from an invalid path due to a
    directory restructuring
-   Fixed profile markers for `SyncPointPlaybackSystem` so that system names are
    displayed correctly
-   Fixed `Child` buffer being left behind on changed parents when using
    Improved or Extreme Transforms

### Improved

-   Fluent Queries now only generates GC when using shared component filters,
    whose support will be removed in 0.7 to provide full Burst-compatibility
-   `MergeBlackboardsSystem` only runs when queries are matched, improving main
    thread performance

## [0.6.0] – 2022-11-16

Officially supports Entities [1.0.0 experimental]

### Added

-   *New Feature:* Added `ICustomEditorBootstrap` which allows modifying the
    setup of an Editor world similar to `ICustomBootstrap`
-   Added new script templates
-   Added `LatiosWorldUnmanaged`, which is now the recommended method of
    accessing Blackboard Entities, Managed Struct Components, Collection
    Components, and the `syncPoint` from unmanaged systems
-   Added methods and extensions to `LatiosWorld`, `SubSystem`, `SuperSystem`,
    `WorldUnmanaged`, `EntityManager`, and `SystemState` for retrieving a
    `LatiosWorldUnmanaged` which can be cached in `OnCreate()` of a system
-   Added `LatiosWorld.zeroToleranceForExceptions` which will cause systems to
    stop updating after a system throws an exception caught by one of the Latios
    Framework’s system update methods, preventing following systems from adding
    false errors to the error log (this is off by default)
-   Added `SyncPointPlaybackSystem.AddMainThreadCompletionForProducer()` to
    specify the command buffer was only used from the main thread

### Changed

-   **Breaking:** `ICustomConversionBootstrap` has been replaced with
    `ICustomBakingBootstrap`
-   **Breaking:** Smart Blobbers have been completely redesigned to work with
    the baking workflow and any code using them will have to be rewritten
-   **Breaking:** Renamed `IManagedComponent` to `IManagedStructComponent` and
    modified its `AssociatedComponentType` to use a `ComponentType`
-   **Breaking:** `ICollectionComponent`’s interface and related APIs have
    changed such that it is now the responsibility of the component itself to
    track if it has been initialized
-   **Breaking:** `ICollectionComponent` must now be unmanaged, and managed data
    should be migrated to a companion `IManagedStructComponent`
-   **Breaking:** Renamed `SuperSystem.GetOrCreateAndAddSystem()` to
    `SuperSystem.GetOrCreateAndAddManagedSystem()`
-   Renamed `ComponentSystemBaseSystemHandleUntypedUnion` to
    `ComponentSystemBaseSystemHandleUnion`
-   Blackboard Entities are now backed by a `LatiosWorldUnmanaged` rather than
    an `EntityManager`
-   Authoring components now use Bakers rather than implement conversion
    interfaces
-   Scene Manager now forces subscenes to load synchronously
-   `DontDestroyOnSceneChangeTag` will now prevent an entity associated with a
    subscene from being destroyed when the subscene is unloaded
-   `LatiosWorld.syncPoint` is now an unmanaged system returned by ref after a
    safety check
-   `SyncPointPlaybackSystem` is now updated by a
    `SyncPointPlaybackSystemDispatch` system type, which should be used for
    attribute-based ordering
-   System ordering in `InitializationSystemGroup` has been altered to work
    without `ConvertToEntitySystem`
-   All component types whose namespace exactly equals “Unity.Entities” are
    excluded from Blackboard Entity merging

### Fixed

-   Fixed determinism of `ParentSystem` when adding `Child` buffer

### Improved

-   All custom collections (including command buffers) no longer use
    `DisposeSentinel` and now support custom allocators
-   Both managed and unmanaged systems are created simultaneously in
    `BootstrapTools.InjectSystems()`, more closely matching Unity’s default
    behavior
-   `CoreBootstrap` now detects install conflicts at runtime
-   `EntityManager` extension methods for Collection Components and Managed
    Struct Components now only require the presence of a `LatiosWorldUnmanaged`
    rather than a full `LatiosWorld`, though using a cached
    `LatiosWorldUnmanaged` is still faster
-   Collection Components and `syncPoint` job dependencies are now automatically
    tracked for all systems updated by a Latios Framework system updater
-   `SubSystem` and `SuperSystem` now only require the presence of a
    `LatiosWorldUnmanaged` rather than a full `LatiosWorld`
-   `ImprovedTransforms` and `ExtremeHierarchy` systems are all fully unmanaged
    and Burst-compiled
-   `SyncPointPlaybackSystem` now uses a `RewindableAllocator` for command
    buffers

### Removed

-   Removed `SystemState` Blackboard Entity accessors, which should now be
    replaced with `LatiosWorldUnmanaged`
-   Removed support for Collection Components and Managed Struct Components in
    `EntityDataCopyKit`
-   Removed some Collection Component `JobHandle` manipulation variants as
    `JobHandle` combining performance quirks in 2022 is not yet fully understood
-   Removed `BlackboardUnmanagedSystem`, which is now replaced by
    `LatiosWorldUnmanaged`

## [0.5.8] – 2022-11-10

Officially supports Entities [0.51.1]

### Fixed

-   Fixed incorrect input locking behavior for batching Smart Blobbers that
    caused exceptions when used with subscenes

## [0.5.7] – 2022-8-28

Officially supports Entities [0.51.1]

### Changed

-   Explicitly applied Script Updater changes for hashmap renaming

## [0.5.6] – 2022-8-21

Officially supports Entities [0.50.1] – [0.51.1]

### Added

-   Added `ObjectAuthoringExtensions.DestroyDuringConversion()` to facilitate
    destroying temporary `UnityEngine.Objects` during conversion that works in
    Editor Mode, Play Mode, and Runtime.

## [0.5.5] – 2022-8-14

Officially supports Entities [0.50.1] – [0.51.1]

### Fixed

-   Fixed Extreme Transforms crashing on startup in builds

## [0.5.2] – 2022-7-3

Officially supports Entities [0.50.1]

### Fixed

-   Fixed Smart Blobbers hiding exception stack traces

## [0.5.1] – 2022-6-19

Officially supports Entities [0.50.1]

### Added

-   Added
    `CustomConversionSettings.OnPostCreateConversionWorldEventWrapper.OnPostCreateConversionWorld`
    which can be used to post-process the conversion world after all systems are
    created

### Fixed

-   Applied the Entities 0.51 `ParentSystem` bugfix to `ImprovedParentSystem`
    and `ExtremeParentSystem`
-   Made the `Child` buffer order deterministic for `ImprovedParentSystem` and
    `ExtremeParentSystem`

## [0.5.0] – 2022-6-13

Officially supports Entities [0.50.1]

### Added

-   *New Feature:* Added experimental NetCode support
-   *New Feature:* Added Smart Blobbers which provide a significantly better
    workflow for Blob Asset conversion
-   *New Feature:* Added `ICustomConversionBootstrap` which allows modifying the
    setup of a conversion world similar to `ICustomBootstrap`
-   *New Feature:* Added Improved Transforms and Extreme Transforms which
    provide faster and more bug-free transform system replacements while still
    preserving the user API
-   Added new bootstraps and templates
-   Exposed `UnsafeParallelBlockList` which is a fast container that can be
    written to in parallel
-   Added `[NoGroupInjection]` attribute to prevent a system from being injected
    in a default group
-   Added `CoreBootstrap` which provides installers for Scene Management and the
    new Transform systems
-   Added chunk component options to Fluent queries
-   Added Fluent method `WithDelegate()` which allows using a custom delegate in
    a Fluent query rather than an extension method
-   Added `LatiosWorld.ForceCreateNewSceneBlackboardEntityAndCallOnNewScene()`
    which provides an alternative to use that functionality without Scene
    Manager installed
-   Added `LatiosMath.RotateExtents()` overload that uses a scalar `float` for
    `extents`
-   Added `BlobBuilderArray.ConstructFromNativeArray<T>()` overload which uses
    raw pointer and length as arguments
-   Added `GetLength()` extension for Blob Assets

### Changed

-   Scene Manager is no longer automatically created by
    `LatiosInitializationSystemGroup` and instead has a dedicated installer
-   All systems inject into the base `InitializationSystemGroup` instead of the
    Latios-prefixed version
-   The ordering of systems in `InitializationSystemGroup` relative to Unity
    systems have been altered

### Fixed

-   Added missing `readOnly` argument to `BlackboardEntity.GetBuffer()`
-   Removed GC allocations in `EntityManager` Collection Component and Managed
    Struct Component implementations
-   Fixed a bug where `SuperSystem` was consuming exceptions incorrectly and
    hiding stack traces
-   Added missing namespace to StringBuilderExtensions

### Improved

-   Added significantly more XML documentation
-   `InstantiateCommandBuffer` can now have up to 15 tags added incrementally

### Removed

-   Removed `[BurstPatcher]` and `[IgnoreBurstPatcher]` attributes

## [0.4.5] – 2022-3-20

Officially supports Entities [0.50.0]

### Added

-   Added `Fluent()` extension method for `SystemState` for use of Fluent in
    `ISystem`.
-   Added `ISystemNewScene` and `ISystemShouldUpdate` which provides additional
    callbacks for `ISystem` matching that of `SubSystem` and `SuperSystem`.
-   Added `GetWorldBlackboardEntity()` and `GetSceneBlackboardEntity()`
    extension methods for `SystemState` which are compatible with Burst.
-   Added `SortingSystemTracker` which can detect systems added or removed from
    a `ComponentSystemGroup`.
-   Added `CopyComponentData()` and `CopyDynamicBuffer()` extension methods for
    `EntityManager` which are compatible with Burst.
-   Added `CopySharedComponent()` extension method for `EntityManager`.
-   Added `GetSystemEnumerator()` extension method for `ComponentSystemGroup`
    which can step through all managed and unmanaged systems in update order.
-   Added `ExecutingSystemHandle()` extension method for `WorldUnmanaged` which
    returns a handle to the system currently in its `OnUpdate()` routine. This
    is compatible with Burst.
-   Added `AsManagedSystem()` extension method for `World` and `WorldUnmanaged`
    which extract the class reference of a managed system from a
    `SystemHandleUntyped`.
-   Added `GetAllSystemStates()` extension method for `WorldUnmanaged` which can
    gather all `SystemState` instances and consequently all systems in a World
    including unmanaged systems.
-   Added `WorldExposedExtensions.GetMetaIdForType()` which can generate an ID
    for a `System.Type` which can be compared against
    `SystemState.UnmanagedMetaIndex`.

### Changed

-   `BootstrapTools.InjectSystem` can now inject unmanaged systems. Many other
    `BootstrapTools` functions now handle unmanaged systems too.

### Fixed

-   Fixed an issue where custom command buffers could not be played back from
    within a Burst context.
-   All custom `ComponentSystemGroup` types including `SuperSystem` and
    `RootSuperSystem` support unmanaged systems.
-   All custom `ComponentSystemGroup` types including `SuperSystem` and
    `RootSuperSystem` support `IRateManager`.
-   All custom `ComponentSystemGroup` types including `SuperSystem` and
    `RootSuperSystem` support automatic system sorting when systems are added
    and removed.
-   Fixed an issue where the managed and collection component reactive systems
    were being stripped in builds.
-   Fixed an issue with `EntityDataCopyKit` not handling default-valued shared
    components correctly.

### Improved

-   `EntityDataCopyKit` no longer uses reflection.
-   All custom `ComponentSystemGroup` types including `SuperSystem` and
    `RootSuperSystem` behave much more like Unity’s systems regarding error
    handling.

## [0.4.3] – 2022-2-21

Officially supports Entities [0.17.0]

### Fixed

-   Fixed GC allocations caused by using a foreach on `IReadOnlyList` type
-   Removed a reference to the Input System package in Latios.Core.asmdef

## [0.4.2] – 2021-10-5

Officially supports Entities [0.17.0]

### Added

-   Added `flags` parameter to `LatiosWorld` constructor which is required for
    NetCode projects.

### Improved

-   Improved the error message for when a `SubSystem` begins its `OnUpdate()`
    procedure but the previous `SubSystem` did not clean up the automatic
    dependency stack. The error now identifies both systems and suggests the
    more likely cause being an exception in the previous `SubSystem`.

## [0.4.1] – 2021-9-16

Officially supports Entities [0.17.0]

### Added

-   Added `simd.length()`.
-   Added `simd.select()` which allows for selecting the x, y, and z components
    using separate masks.
-   Added `simd.cminxyz()` and `cmaxxyz()` which computes the min/max between x,
    y, and z for each of the four float3 values.
-   Added `simd.abs()`.
-   Added `simd.project()`.
-   Added `simd.isfiniteallxyz()` which returns true for each float3 if the x,
    y, and z components are all finite values.

## [0.4.0] – 2021-8-9

Officially supports Entities [0.17.0]

### Added

-   Added argument `isInitialized` to
    `BlackboardEntity.AddCollectionComponent()`
-   *New Feature:* Added `OnNewScene()` callback to `SubSystem` and
    `SuperSystem`, which can be used to initialize components on the
    `sceneBlackboardEntity`.
-   *New Feature:* Added `EntityWith<T>` and `EntityWithBuffer<T>` for strongly
    typed entity references and left-to-right dereferencing chains.
-   Added `LatiosMath.ComplexMul()`.
-   *New Feature:* Added `Rng` and `RngToolkit` which provide a new framework
    for fast, deterministic, parallel random number generation.

### Improved

-   Added safety checks for improper usage of `sceneBlackboardEntity` that would
    only cause issues in builds.

## [0.3.1] – 2021-3-7

Officially supports Entities [0.17.0]

### Fixed

-   Fixed several compiler errors when creating builds
-   Removed dangling references to the Code Analysis package in
    Latios.Core.Editor.asmdef

## [0.3.0] – 2021-3-4

Officially supports Entities [0.17.0]

### Added

-   **Breaking**: Added `LatiosWorld.useExplicitSystemOrdering`. You must now
    set this value to true in an `ICustomBootstrap` before injecting additional
    systems.
-   Added the following convenience properties to `LatiosWorld`:
    `initializationSystemGroup`, `simulationsSystemGroup`,
    `presentationSystemGroup `
-   *New Feature:* Added `GameObjectConversionConfigurationSystem` which when
    subclassed allows for customizing the conversion world before any other
    systems run. Not all entities have been gathered when `OnUpdate()` is
    called. Note that this mechanism is likely temporary and may change after a
    Unity bug is fixed.
-   *New Feature:* Added the following containers: `EnableCommandBuffer`,
    `DisableCommandBuffer`, `DestroyCommandBuffer`, `InstantiateCommandBuffer`,
    `EntityOperationCommandBuffer`
-   *New Feature:* Added `SyncPointPlaybackSystem` which can play back
    `EntityCommandBuffers` as well as the new command buffer types
-   *New Feature:* Added `LatiosWorld.syncPoint` which provides fast access to
    the `SyncPointPlaybackSystem` and removes the need to call
    `AddJobHandleForProducer()` when invoked from a `SubSystem`
-   Added `BlackboardEntity.AddComponentDataIfMissing<T>(T data)` which systems
    can use to default-initialize `worldBlackboardEntity` components in
    `OnCreate()` without overriding custom settings
-   Added `PreSyncPointGroup` for scheduling non-ECS jobs to execute on worker
    threads during the sync point
-   Added extension `BlobBuilder.ConstructFromNativeArray<T>()` which allows
    initializing a `BlobArray<T>` from a `NativeArray<T>`
-   Added extension `BlobBuilder.AllocateFixedString<T>()` which allows
    initializing a `BlobString` from a `FixedString###` or `HeapString` inside a
    Burst job
-   Added extension `NativeList<T>.AddRangeFromBlob()` which appends the
    `BlobArray` data to the list
-   Added extension `StringBuilder.Append()` which works with `FixedString###`
    hi
-   Added new assembly `Entities.Exposed` which uses the namespace
    `Unity.Entities.Exposed` and extends the API with the following:
    -   `EntityLocationInChunk` – a new struct type
    -   Extension method `EntityManager.GetEntityLocationInChunk()` used for
        advanced algorithms
    -   Extension method `World.ExecutingSystemType()` used for profiling
-   Added new assembly `MathematicsExpansion` which extends the
    `Unity.Mathematics.math` API with `select()` functions for `bool` types
-   `Added lots of new API for simdFloat3`

### `Changed`

-   **Breaking:** Global Entities have been renamed to Blackboard Entities. See
    upgrade guide for details.
-   `TransfomUniformScalePatchConversionSystem` was moved from
    `Latios.Authoring` to `Latios.Authoring.Systems`
-   **Breaking:** Renamed `LatiosSyncPointGroup` to `LatiosWorldSyncGroup`
-   **Breaking:** `LatiosInitializationSystemGroup`,
    `LatiosSimulationSystemGroup`, `LatiosPresentationSystemGroup`,
    `LatiosFixedSimulationSystemGroup`, `PreSyncPointGroup`, and
    `LatiosWorldSyncGroup` all use `SuperSystem` updating behavior.
    `SuperSystem` and `SubSystem` types added to one of these groups will now
    have `ShouldUpdateSystem()` invoked. Unfortunately, this also means that
    `ISystemBase` systems added to these groups will not function. `ISystemBase`
    is advised against in the short-term. But if you must have it, put it inside
    a custom `ComponentSystemGroup` instead.

### Removed

-   BurstPatcher has been removed.

### Improved

-   Systems created in `LatiosWorld` now use attribute-based ordering which may
    improve compatibility with more DOTS projects
-   All authoring components have now been categorized under *Latios-\>Core* in
    the *Add Component* menu
-   All authoring components now have public access so that custom scripts can
    reference them
-   **Breaking:** `BlackboardEntity` methods have been revised to more closely
    resemble `EntityManager` methods
-   Added an additional argument to `BootstrapTools.InjectUnitySystems()` which
    silences warnings and made this silencing behavior default
-   Setting `DestroyEntitiesOnSceneChangeSystem.Enabled` to `false` disables its
    functionality

## [0.2.2] - 2021-1-24

Officially supports Entities [0.17.0]

### Changed

-   BurstPatcher has been deprecated. If you still need the functionality, add
    `BURST_PATCHER` to the Scripting Define Symbols

### Fixed

-   Fixed a `NullReferenceException` in the *Bootstrap – Injection Workflow*
    template

## [0.2.1] - 2020-10-31

Officially supports Entities [0.16.0]

### Changed

-   Due to Entities changes, modifying `SuperSystem`s or
    `LatiosInitializationSystemGroup` after bootstrap is no longer supported

### Fixed

-   Fixed some deprecation warnings

## [0.2.0] - 2020-9-22

Officially supports Entities [0.14.0]

### Added

-   *New Feature*: Burst Patcher uses codegen at Build time to allow for
    compilation of generic jobs scheduled by generic methods even if the generic
    argument or the job is private
-   Added "Bootstrap - Explicit Workflow" creation tool
-   Added `BootstrapTools.InjectUnitySystems` which does exactly what it sounds
    like
-   Added `BootstrapTools.UpdatePlayerLoopWithDelayedSimulation` which runs
    `SimulationSystemGroup` after rendering
-   *New Feature*: Collection component dependency management now hooks into the
    `Dependency` field that lambda jobs use
-   *New Feature*: Fluent Queries provide a fluent extendible shorthand for
    generating `EntityQuery` instances
-   Added `LatiosSimulationSystemGroup` and `LatiosPresentationSystemGroup`
-   Added `IManagedComponent` and `ICollectionComponent` API to `ManagedEntity`
-   Added float2 variants for `SmoothStart` and `SmoothStop` in `LatiosMath`

### Changed

-   The original Bootstrap creation is now labeled "Bootstrap - Injection
    Workflow"
-   Renamed `IComponent` to `IManagedComponent`
-   `IManagedComponent` and `ICollectionComponent` behave differently due to
    DOTS evolution. See the documentation for how to use them.
-   `FixedString` is used instead of `NativeString`
-   `BootstrapTools.UpdatePlayerLoopWithFixedUpdate` is now
    `BootstrapTools.AddWorldToCurrentPlayerLoopWithFixedUpdate` with updated API
    to mirror `ScriptBehaviorUpdateOrder`
-   `EntityDataCopyKit` now ignores collection components by default
-   `SubSystem` now subclasses `SystemBase` and should be used for all logical
    systems
-   `LatiosInitializationSystemGroup` was reworked for compatibility with the
    latest Entities

### Removed

-   `JobSubSystem` - use `SubSystem` instead which now subclasses `SystemBase`

### Fixed

-   Fixed an issue with `EntityDataCopyKit` and dynamic buffers
-   `SuperSystem` and `RootSuperSystem` should now properly invoke
    `OnStopRunning` of children systems on failure of `ShouldUpdateSystem`
-   Fixed `IManagedComponent` and `ICollectionComponent` reactive systems not
    reacting aside from the first and last frames of a scene
-   `SceneManagerSystem` now properly removes `RequestLoadScene` components
    after processing them

### Improved

-   Many exceptions are now wrapped in a
    `[Conditional("ENABLE_UNITY_COLLECTION_CHECKS")]`
-   Entity destruction on scene changes now works immediately instead of delayed
    unload

## [0.1.0] - 2019-12-21

### This is the first release of *Latios Core for DOTS*.
