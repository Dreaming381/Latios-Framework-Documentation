# Changelog

All notable changes to this package will be documented in this file.

The format is based on [Keep a Changelog](http://keepachangelog.com/en/1.0.0/)
and this project adheres to [Semantic
Versioning](http://semver.org/spec/v2.0.0.html).

## [0.15.0] – 2026-4-12

Officially supports Entities [1.4.4]

### Added

-   Added new `qvvs` APIs which automatically normalize quaternions
-   Added `AuthoringSiblingIndex`, `BakedLocalTransformOverride`, and
    `BakedParentOverride` to improve transform baking experience for both QVVS
    and Unity Transforms
-   Added new API to `TransformAspect` to provide full hierarchy access

### Changed

-   **Breaking:** Completely replaced all ECS components, systems, and utility
    APIs related to hierarchy management
-   **Breaking:** `TransformAspect` is no longer a `Unity.Entities.IAspect`, and
    must be obtained via special APIs
-   **Breaking:** Renamed `HierarchyUpdateMode.Flags` to `InheritanceFlags`,
    which now includes `CopyParent` to replace `CopyParentWorldTransformTag`,
    and also changed baking APIs appropriately
-   **Breaking:** `WorldTransformReadOnlyAspect` is no longer a
    `Unity.Entities.IAspect`, and now has enhanced implementations for much of
    the code it used to generate, removing the need for the separate
    `WorldTransformReadOnlyTypeHandle`
-   **Breaking:** Renamed `TransformQvvs.worldIndex` to `context32` and all
    associated API to better represent that this is a context-dependent 32-bit
    value
-   **Breaking:** Replaced `PostTransformSuperSystem` with
    `ExportToGameObjectTransformsEndInitializationSuperSystem` and
    `ExportToGameObjectTransformsEndSimulationSuperSystem`
-   **Breaking:** Changed or removed `TransformAspect` API that involved parent
    access to reflect the new hierarchy paradigm
-   **Breaking:** Changed the API for accessing `TransformAspect` from
    `ComponentBroker`
-   **Breaking:** `TransformsBootstrap.InstallTransforms()` now only has a
    single parameter for the `LatiosWorld`
-   When QVVS Transforms are used, Unity’s
    `BakingOnlyEntityAuthoringBakingSystem` is forced to update at the beginning
    of `TransformBakingSystemGroup` rather than in the middle of the default
    group
-   Writing to a `TransformAspect` when the inheritance mode is `CopyParent` is
    now silently ignored rather than throwing an exception

### Improved

-   Hierarchy propagation will now ensure rotation quaternions are normalized
    for improved robustness

### Removed

-   **Breaking:** Removed all Abstract aspects except
    `WorldTransformReadOnlyAspect`
-   **Breaking:** Removed `CopyTransformToEntity` for `GameObjectEntity`, as the
    appropriate time for this is no longer understood

## [0.14.8] – 2026-1-3

Officially supports Entities [1.4.3]

### Added

-   Added `Equals()` method to `TransformQvvs` which performs an exact bit match

## [0.14.6] – 2025-12-20

Officially supports Entities [1.4.3]

This release only contains internal compatibility changes

## [0.14.0] – 2025-10-18

Officially supports Entities [1.3.14]

### Added

-   Added `GameObjectEntityUnmanaged` component which is now attached to any
    `GameObjectEntity`, allowing for unmanaged access to the
    `UnityObjectRef<GameObject>`
-   Added LATIOS_BURST_DETERMINISM support

## [0.13.0] – 2025-7-6

Officially supports Entities [1.3.14]

### Added

-   Added `WorldTransformReadOnlyAspect` method `Has()` which checks if the
    underlying world transform type is in the specified `ArchetypeChunk`
-   Added `FluentQuery` extension `WithoutWorldTransform()` to exclude the
    underlying world transform type
-   Added `enableTransformSyncing` to `GameObjectEntityAuthoring` which when
    disabled may prevent unnecessary transform sync jobs from executing (that
    the engine might sync on)

### Changed

-   **Breaking:** `MotionHistoryUpdateSuperSystem` now updates at the beginning
    of `SimulationSystemGroup` and there is a new
    `MotionHistoryInitializeSuperSystem` which updates within
    `PostSyncPointGroup` instead
-   A second transform system update is performed in `PostSyncPointGroup` prior
    to `MotionHistoryInitializeSuperSystem` when Qvvs Transforms are installed

### Fixed

-   Fixed compilation issues in ShaderLibrary hlsl files
-   Fixed missing safety attribute in `CopyGameObjectTransformToEntitySystem`
    when using Unity Transforms

### Removed

-   Removed `version` and `isInitialized` from `PreviousTransform` and
    `TwoAgoTransform`, as Motion History no longer uses the `worldIndex`
    property of QVVS Transforms and now propagates the property unaltered

## [0.12.6] – 2025-4-20

Officially supports Entities [1.3.14]

### Added

-   Added `forwardDirection`, `rightDirection`, and `upDirection` properties to
    `TransformQvvs`

### Fixed

-   The new entities update fixes an issue with `GameObjectEntityHost` as a
    child of a `GameObject` with `LinkedEntityGroupAuthoring`, and will no
    longer result in errors upon subscene unload

## [0.12.0] – 2025-2-23

Officially supports Entities [1.3.9]

### Added

-   *New Feature:* Added shader library for `TransformQvvs` operations
-   Added `[System.Serializable]` attribute to `TransformQvvs` and
    `TransformQvs`

### Changed

-   **Breaking:** Fixed API typo in `IBaker` extension method from
    `AddHiearchyModeFlags()` to `AddHierarchyModeFlags()`
-   Moved `MotionHistoryUpdateSuperSystem` to `PostSyncPointGroup`

## [0.11.0] – 2024-9-29

Officially supports Entities [1.3.2]

### Added

-   *New Feature:* Added NetCode default ghost variant for `WorldTransform`
-   Added NetCode-exclusive aliased fields for `WorldTransform` so that new
    ghost variants can be added for it
-   Added experimental `DefaultWorldTransformSmoothingAction` which can be
    enabled via the `Register()` method

## [0.10.0] – 2024-5-26

Officially supports Entities [1.2.1]

### Added

-   *New Feature:* Added GameObjectEntity support for Unity Transforms, which
    can be installed via
    `TransformBootstrap.InstallGameObjectEntitySynchronization()`

### Fixed

-   Fixed an issue with `Transform` extension method `GetQvvsRelativeTo()`
    sometime producing the wrong result
-   Corrected wrong namespace for GameObjectEntity systems

### Improved

-   Cached QVVS `CompanionGameObjectUpdateTransformSystem` is now Burst-compiled

### Removed

-   Removed Cached QVVS `CompanionGameObjectUpdateTransformSystem`, as the Unity
    implementation is sufficient

## [0.9.0] – 2024-1-20

Officially supports Entities [1.1.0-pre.3]

### Changed

-   **Breaking:** Renamed `FluentQuery` extension
    `WithWorldTransformReadOnlyWeak()` to `WithWorldTransformReadOnly()`
-   When using Unity Transforms,
    `WorldTransformReadOnlyAspect.worldTransformQvvs` now uses the `x` component
    of the nonlinear scale extracted from the `LocalToWorld` matrix as the QVVS
    uniform scale

## [0.8.1] – 2023-10-2

Officially supports Entities [1.0.16]

### Fixed

-   Fixed Companion Game Objects being disabled

## [0.8.0] – 2023-9-23

Officially supports Entities [1.0.16]

### Added

-   *New Feature:* Added Unity Transforms support via the
    LAITOS_TRANSFORMS_UNITY scripting define
-   *New Feature:* Added `HierarchyUpdateMode` and associated baking APIs for
    locking world-space transform values when the hierarchy updates, choosing to
    update `LocalTransform` for such values instead
-   *New Feature:* Added GameObjectEntity which allows setting up a Game Object
    outside of a subscene as an entity tied to the original Game Object, with
    that entity optionally being an entity inside a subscene
-   Added `Abstract` namespaces containing `IAspect` and helper static classes
    for abstracting the transform system used in common cases
-   Added `HybridTransformsSyncPointSuperSystem` which is when GameObjectEntity
    bindings occur

### Changed

-   `TransformBakeUtils.GetQvvsRelativeTo()` now returns
    `TransformQvvs.identity` if both `Transform` arguments are actually the same
    `Transform`

### Improved

-   Made the `TransformQvs` argument in `qvvs.mul()` an `in` parameter

## [0.7.0] – 2023-5-29

Officially supports Entities [1.0.10]

### This is the first release of *QVVS Transforms*.
