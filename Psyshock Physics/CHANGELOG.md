# Changelog

All notable changes to this package will be documented in this file.

The format is based on [Keep a Changelog](http://keepachangelog.com/en/1.0.0/)
and this project adheres to [Semantic
Versioning](http://semver.org/spec/v2.0.0.html).

## [0.11.1] – 2024-10-5

Officially supports Entities [1.3.2]

### Added

-   Added `IBaker` extension methods `ShouldBakeSingleCollider()` and
    `GetMultiColliderBakeMode()` to help identify baking rules when multiple
    authoring colliders exist on a single `GameObject`, or when such colliders
    are referenced by a parent `CustomColliderAuthoring`

### Fixed

-   Fixed `PairStream.ConcatenateFrom()` memory stomps
-   Fixed `[WriteOnly]` safety error when attempting to read `PairStream.Pair`
    properties created from a `PairStream.ParallelWriter`

## [0.11.0] – 2024-9-29

Officially supports Entities [1.3.2]

### Added

-   *New Feature:* Added multi-layer queries which can be provided via a
    `ReadOnlySpan<CollisionLayer>` and the corresponding layer can be obtained
    via `LayerBodyInfo.layerIndex`
-   Added multi-layer `FindObjects` for `foreach` enumeration

## [0.10.7] – 2024-8-25

Officially supports Entities [1.2.1] – [1.2.4]

### Fixed

-   Fixed FindPairs repeating results when using only a single cell in the
    `CollisionLayer`
-   Fixed various queries with TriMesh colliders that have scaling

## [0.10.4] – 2024-7-20

Officially supports Entities [1.2.1] – [1.2.3]

### Fixed

-   Fixed false negatives in `Physics.FindObjects()` used in a `foreach`

## [0.10.2] – 2024-6-22

Officially supports Entities [1.2.1] – [1.2.3]

### Fixed

-   Added guard against non-finite values for `constraintTau` in
    `UnitySim.ConstraintTauAndDampingFrom()`

## [0.10.1] – 2024-6-9

Officially supports Entities [1.2.1]

### Added

-   Added `CollisionLayerSettings.kDefault` as an alternative to
    `BuildCollisionLayerConfig.defaultSettings`

### Changed

-   `Physics.DistaneBetween()` for pairs of box colliders now uses GJK and EPA
    instead of the old SAT algorithm, as the old algorithm proved to be
    unreliable and needs to be rewritten
-   Collision layers using subdivisions per axis of (1, 1, 1) no longer allocate
    a cross-bucket, and all FindPairs and ForEachPairs operations that use a
    parallel schedule command on such a layer will instead use ScheduleSingle()

### Fixed

-   Fixed Burst compatibility of `UnitySim` stiff spring constants
-   Fixed contact normal being in opposite direction in
    `UnitySim.ContactsBetween()` between pairs involving boxes and triangles
-   Fixed normal vector determination in `Physics.DistanceBetween()` for pairs
    which utilize GJK
-   Fixed ForEachPair alias detection not executing with safety checks enabled
-   Fixed memory aliasing of PairStream when writing elements using a
    ParallelWriterKey with a cross-bucket element marked read-write and a cell
    element marked read-only

### Improved

-   Added various new safety check validations across the codebase, allowing for
    automatic detection of more user mistakes

## [0.10.0] – 2024-5-26

Officially supports Entities [1.2.1]

### Added

-   *New Feature:* Added many new APIs and functionality in UnitySim to support
    rigid body simulation, which is now possible to do in Psyshock
-   *New Feature:* Added `Physics.ForEachPair`, `IForEachPairProcessor`,
    `PairStream`, and related APIs in FindPairs which allow for capturing
    arbitrary data about pairs and iterating over them in parallel, intended for
    use in constraint solvers
-   Added `Physics.BuildCollisionLayer()` overloads which accept `NativeList`
    arguments
-   Added default interface methods `Begin()` and `End()` to
    `IDistanceBetweenAllProcessor`, which provide the number of sub-colliders
    for a given pair
-   Added FindPairs `ScheduleParallelByA()` which is a scheduling mode that only
    guarantees thread-safety for the ‘A’ body in a bipartite (dual layer)
    operation, but does so using a single parallel job
-   Added center of mass and inertia tensor properties to convex, tri-mesh, and
    compound collider blob assets
-   Added `CollisionLayerBucketIndexCalculator` which allows you to predict the
    bucket index of an `Aabb` for adding pairs to `PairStream`
-   Added entity property to `SafeEntity` to simplify syntax over a cast
    operation in some contexts
-   Added `DistanceBetweenAllCache` which can be used as a simple
    `IDistanceBetweenAllProcessor`

### Changed

-   **Breaking:** `CollisionLayer` is now backed by `NativeLists` instead of
    `NativeArrays`, meaning that the element count of a `CollisionLayer` is no
    longer always safe to access
-   **Breaking:** `IFindPairsProcessor.BeginBucket()` now returns a `bool` which
    when `false` will cause processing of the bucket pair and the `EndBucket()`
    calls to be skipped
-   Safety validation of FindPairs has changed, such that unsafe contexts like
    `ScheduleParallelUnsafe()` will no longer produce valid `SafeEntity`
    instances for use in `PhysicsComponentLookup` and `PhysicsBufferLookup`
-   `WithCrossCache()` is no longer supported in FindPairs using
    `ScheduleParallel()` and will be ignored with a warning

### Fixed

-   Fixed a memory corruption caused by a defensive copy made to blob asset
    collider types
-   Fixed `PhysicsDebug.DrawCollider()` not transforming a sphere’s center
    correctly
-   Fixed various issues with `UnitySim.ContactsBetween()`
-   Fixed `Physics.DistanceBetween()` not correctly identifying the correct
    feature normals for some pairs involving triangle and convex colliders
-   Fixed `maxDistance` not being accounted for in `Physics.DistanceBetween()`
    and `Physics.DistanceBetweenAll()` involving tri-mesh colliders
-   Fixed wrong transform being used in `Physics.DistanceBetween()` involving a
    box and a triangle
-   Fixed `Physics.DistanceBetweenAll()` returning extra invalid results due to
    a missing condition guard in the dispatcher
-   Fixed excessive allocation in CompoundColliderBlob
-   `CollisionLayer.Dispose()` taking a `JobHandle` no longer forces job
    completion if allocated with a custom allocator

### Improved

-   Added an early-out in `Physics.DistanceBetween()` involving a box and a
    triangle, which results in a significant speedup of
    `Physics.DistanceBetween()` and `Physics.DistanceBetweenAll()` for some
    scenarios involving a box and a tri-mesh
-   Improved IL2CPP stripping support for `IFindPairsProcessor` and
    `IFindObjectsProcessor`
-   FindPairs now compiles significantly fewer jobs and provides better profiler
    markers
-   FindPairs is significantly faster when there are a large number of colliders
    in a cross-bucket while testing against cells
-   Improved performance of layer queries when running without Burst
-   `Physics.BuildCollisionLayer()` no longer induces sync points when the
    passed in query contains enabled component filtering

## [0.9.4] – 2024-3-16

Officially supports Entities [1.1.0-pre.3]

### Fixed

-   Fixed range exception in `UnitySim.ContactsBetween()`

## [0.9.1] – 2024-1-27

Officially supports Entities [1.1.0-pre.3]

### Fixed

-   Fixed FindPairs in builds that compiled for targets other than AVX

## [0.9.0] – 2024-1-20

Officially supports Entities [1.1.0-pre.3]

### Added

-   *New Feature:* Added `UnitySim.ContactsBetween()` which can find a contact
    manifold from a `ColliderDistanceResult` using an algorithm that behaves
    similar to Unity Physics
-   *New Feature:* Added `XpbdSim` methods `SolveDistanceConstraint()` and
    `SolveVolumeConstraint()` for use in XPBD particle simulations
-   Added `ColliderDistanceResult` methods `FlipInPlace()` and `ToFlipped()` for
    switching the values of A and B in the result

### Changed

-   **Breaking:** `ConvexColliderBlob` was redesigned with new arrays and shrunk
    data sizes of existing indices arrays
-   Field layouts of all collider types have been reworked to shrink `Collider`
    down to 48 bytes

### Improved

-   FindPairs now uses AVX on supported platforms for a mild performance
    improvement
-   Reworked convex collider point and ray queries to use 2D hulls with stretch
    along one axis is zero

## [0.8.3] – 2023-10-15

Officially supports Entities [1.0.16]

### Added

-   Added CollisionLayer.CreateEmptyCollisionLayer() as a cheaper and
    less-boilerplate option for creating an empty CollisionLayer with given
    settings

## [0.8.0] – 2023-9-23

Officially supports Entities [1.0.16]

### Added

-   *New Feature*: Added TriMesh collider type with baking support by a
    non-convex Mesh Collider component
-   *New Feature*: Added Unity Transforms (Entities 1.0) support
-   Added `UnitySim` namespace with struct `ContactsBetweenResult`, although
    this currently serves no purpose
-   Added `Physics.DistanceBetweenAll()` which is able to dispatch a result to a
    processor for each sub-collider pair
-   Added an enumerable version of `Physics.FindObjects()` which can be used in
    a `foreach` expression, bypassing the need for a dedicated
    `IFindObjectsProcessor`
-   Added `TransformAabb()` overload which takes a
    `WorldTransformReadOnlyAspect`

### Changed

-   **Breaking:** Renamed `IBaker.RequestCreateBlobAsset(Mesh mesh)` to
    `RequestCreateConvexBlobAsset` to help differentiate it from the new TriMesh
    equivalent that takes the same argument
-   Renamed `ConvexColliderBaker` to `MeshColliderBaker` which will bake both
    Convex and TriMesh colliders given an authoring Mesh Collider component
-   The sizes of some query result types have increased to accommodate future
    simulation efforts

### Fixed

-   Fixed baking multiple colliders on the same authoring Game Object as a
    compound having the wrong child collider transforms
-   Fixed reported hitpoint and normal when a sphere or capsule center perfectly
    touches the closest feature of the other non-sphere collider in a
    `DistanceBetween()` operation
-   Fixed returned normals in queries involving convex colliders

### Improved

-   `FindObjectsResult.layer` now returns by `ref readonly` for improved
    performance

## [0.7.7] – 2023-8-15

Officially supports Entities [1.0.14]

### Added

-   Added default interface methods `BeginBucket()` and `EndBucket()` to
    `IFindPairProcessor` which each run once for every job index
-   Added `Physics.FindPairsJobIndexCount()` methods which return the number of
    job indices used by a FindPairs operation and whose value is always one
    greater than the largest `jobIndex` provided
-   Added `Ray.TransformRay()` overload which accepts a `TransformQvvs`
    transform

### Fixed

-   Fixed casting of `TriangleCollider` to `Collider` retyping the collider as a
    `BoxCollider`
-   Fixed `Physics.BuildCollisionLayer.RunImmediate()` attempting to use
    override AABBs when none were provided
-   Fixed `FindObjectsResult` providing the wrong `bodyIndex`

### Improved

-   Convex collider meshes no longer require being marked as Read/Write in the
    import settings in order to bake properly
-   `Ray.TransformRay()` now uses `in` parameters

## [0.7.3] – 2023-6-10

Officially supports Entities [1.0.10]

### Added

-   Added `Physics.Substep()` which should be used in a foreach statement to
    perform multiple substep iterations for a subdivided `deltaTime`
-   Added `GetRW()` to `PhysicsComponentLookup` to improve performance of
    read-modify-write operations inside FindPairs
-   Added `PhysicsTransformAspectLookup` to modify transforms inside FindPairs

## [0.7.1] – 2023-6-3

Officially supports Entities [1.0.10]

### Fixed

-   Fixed compile errors when using LATIOS_TRANSFORMS_UNITY scripting define
    symbol

## [0.7.0] – 2023-5-29

Officially supports Entities [1.0.10]

### Added

-   Added `StretchMode` to sphere, capsule, and compound colliders for
    specifying how to cope with non-uniform scale, though the defaults usually
    exhibit the desired behavior
-   Added `PhysicsDebug.LogDistanceBetween()` for submitting distance query bug
    reports
-   `CollisionLayer` now stores `sourceIndices`, replacing the `remapSrcIndices`
    APIs
-   Added `Physics.TransformAabb()` overload which accepts a `TransformQvvs`
    transform
-   Added source index properties to `FindPairsResult` and `FindObjectsResult`
    for accessing the original index of the body in question relative to the
    layer’s source `EntityQuery` or `ColliderBody` array
-   Added `CollisionLayer.GetAabb()` for getting the `Aabb` of a specific
    collider
-   Added `IsEnabled` and `SetEnabled` to `PhysicsComponentLookup` and
    `PhysicsBufferLookup` for working with `IEnableableComponent`
-   Added `sourceIndex` to `layerBodyInfo` which corresponds to the
    `sourceIndex` property of the underlying `FindObjects` operation

### Changed

-   **Breaking:** Psyshock now use QVVS Transforms
-   **Breaking:** Psyshock now uses `TransformQvvs` instead of `RigidTransform`
    for all APIs, which means colliders scale automatically with transforms and
    are no longer baked
-   **Breaking:** Renamed `InstallLegacyColliderBakers` to
    `InstallUnityColliderBakers` in `PsyshockBakingBootstrap` as Unity Engine
    authoring components are here to stay
-   **Breaking:** Replaced `Physics.ScaleCollider()` with
    `Physics.ScaleStretchCollider()`, although you will rarely need to call this
    anymore
-   **Breaking:** Renamed `FindPairsResult` `indexA` and `indexB` to
    `bodyIndexA` and `bodyIndexB`
-   **Breaking:** Renamed `CollisionLayer` `Count` and `BucketCount` to `count`
    and `bucketCount`
-   **Breaking:** Merged `ColliderCastResult` `hitpointOnCaster` and
    `hitpointOnTarget` to just a single `hitpoint`
-   Renamed all bakers to no longer have the “Legacy” prefix
-   `BuildCollisionLayer` now requires the presence of `WorldTransform` when
    building from Entity Queries

### Fixed

-   Fixed smart baker types not being marked `[TemporaryBakingType]`
-   Fixed `NullReferenceException` being thrown when baking a convex blob and
    the shared mesh does not have a name

### Improved

-   Improved XML documentation coverage
-   Refactored the internal query source code to be classified by query type and
    collider types rather than API accessibility layers
-   `FindPairs` and `FindObjects` can now schedule jobs from Burst-compiled
    `ISystem`
-   Improved the performance and code size of `FindObjects` and layer queries on
    small Collision Layers
-   `FindObjects` `RunImmediate()` now returns a copy of the processor object
    that was dispatched to, such that results can be stored without pointers or
    other containers

### Removed

-   Removed `PhysicsScale` as scaling can now be extracted from QVVS Transforms
    automatically
-   Removed most `Physics` APIs which take specialized collider types as
    arguments, as these cluttered the public API
-   Removed `WithSourceIndices()` and related APIs from `BuildCollisionLayer()`
    as they are now always included inside the `CollisionLayer` directly

## [0.6.5] – 2023-2-18

Officially supports Entities [1.0.0 prerelease 15]

### Added

-   Added `CollisionLayer.colliderBodies `which provides access to a read-only
    array of` ColliderBody` stored in the `CollisionLayer`
-   Added `Update()` methods to `PhysicsComponentLookup` and
    `PhysicsBufferLookup`

### Fixed

-   Fixed `ConvexColliderSmartBlobberSystem` safety exceptions

### Improved

-   Backported XML documentation for various APIs from the experimental 0.7
    development project

## [0.6.4] – 2022-12-29

Officially supports Entities [1.0.0 prerelease 15]

### Fixed

-   Fixed raycasting a compound collider generating a result in the compound’s
    local space rather than world space
-   Fixed raycasting a compound collider not respecting the compound’s scale
    value
-   Fixed `CollisionLayer.Dispose()` when allocated with a custom allocator

### Improved

-   `SafeEntity` now remaps an `Entity` with `Index` 0 correctly when safety
    checks are enabled

## [0.6.3] – 2022-12-19

Officially supports Entities [1.0.0 prerelease 15]

### Fixed

-   Fixed `Run()` and `Schedule()` for `BuildCollisionLayer()` overload that
    takes an `EntityQuery`, which were missing the enableable component base
    indices array

## [0.6.1] – 2022-11-28

Officially supports Entities [1.0.0 prerelease 15]

This release only contains internal compatibility changes

## [0.6.0] – 2022-11-16

Officially supports Entities [1.0.0 experimental]

### Added

-   *New Feature:* Added `Physics.FindObjects()` which allows for querying a
    single `Aabb` against a `CollisionLayer` in less than O(n)
-   *New Feature:* Added `Physics.Raycast()`, `Physics.ColliderCast()`, and
    `Physics.DistanceBetween()` overloads as well as `Any` counterparts which
    all perform queries against a `CollisionLayer`.
-   Added `CompoundColliderBlob` Smart Blobber
-   Added support for multiple `Collider` authoring components to be attached to
    a Game Object, and such components will be baked into a `CompoundCollider`
-   Added `PhysicsDebug.LogBucketCountsForLayer()` for tuning the distribution
    of cells via `CollisionLayerSettings`
-   Added `BuildCollisionLayerTypeHandles` which can cache the type handles used
    for building a `CollisionLayer` inside `OnCreate()` and later used in
    `OnUpdate()` in an `ISystem`
-   Added `AabbFrom()` overload which accepts a Ray
-   Added `AabbFrom()` overloads which compute bounds around a casted collider
-   Added layer, collider, AABB, and transform properties to `FindPairsResult`
-   Added `WithCrossCache()` extension to FindPairs fluent chain which finds
    more pairs in parallel, but performs extra write and read operations to
    dispatch them safely
-   Added `Raycast()` overloads which accept `start` and `end` arguments instead
    of a `Ray` argument
-   Added `PhysicsComponentLookup.TryGetComponent()`

### Changed

-   **Breaking:** `IFindPairsProcessor` now requires the `Execute()` method pass
    the `FindPairsResult` as an `in` parameter
-   **Breaking:** Authoring has been completely rewritten to use baking workflow
-   **Breaking:**
    `PsyshockConversionBootstrap.InstallLegacyColliderConversion()` has been
    replaced with `PsyshockBakingBootstrap.InstallLegacyColliderBakers()`
-   **Breaking:** Renamed `CollisionLayerSettings.worldAABB` to
    `CollisionLayerSettings.worldAabb`
-   **Breaking:** Renamed `PhysicsComponentDataFromEntity` to
    `PhysicsComponentLookup` and `PhysicsBufferFromEntity` to
    `PhysicsBufferLookup`
-   `Aabb`s with `NaN` values are now stored in a dedicated `NaN` bucket inside
    a `CollisionLayer`
-   `Physics.FindPairs()` now returns a different config object type depending
    on which overload is used, reducing the number of Jobs Burst will compile by
    half
-   `FindPairsResult` uses properties instead of fields for all data
-   Baking will now skip authoring colliders which are disabled

### Fixed

-   `Physics.ScaleCollider()` overload that takes a `Collider` will now dispatch
    Triangle and Convex colliders

### Improved

-   The Collider type’s internal structure was reworked to improve cache loading
    performance
-   Many methods now accept `in` parameters which can improve performance in
    some situations.
-   `BuildCollisionLayer` and other Psyshock API now support custom allocators

### Removed

-   Removed `DontConvertColliderTag`
-   Removed extension methods for `SystemBase` for retrieving a
    `PhysicsComponentDataFromEntity`, as the implicit operator is sufficient for
    all use cases

## [0.5.7] – 2022-8-28

Officially supports Entities [0.51.1]

### Changed

-   Explicitly applied Script Updater changes for hashmap renaming

## [0.5.2] – 2022-7-3

Officially supports Entities [0.50.1]

### Fixed

-   Fixed `FindPairsResult` `bodyAIndex` and `bodyBIndex` which did not generate
    correct indices

### Improved

-   FindPairs performance has been improved after a regression introduced in
    Burst 1.6

## [0.5.0] – 2022-6-13

Officially supports Entities [0.50.1]

### Added

-   *New Feature:* Added convex colliders which use a blob asset for their hull
    but can be non-uniformly scaled at runtime
-   *New Feature:* Added experimental triangle colliders which consist of three
    points
-   Added `PhysicsDebug.DrawCollider()` which draws out the collider shape using
    a configurable resolution

### Fixed

-   Renamed `normalOnSweep` to `normalOnCaster` for `ColliderCastResult` which
    was the intended name

### Improved

-   Legacy collider conversion is now controlled by an installer

## [0.4.5] – 2022-3-20

Officially supports Entities [0.50.0]

### Fixed

-   Fixed an issue where FindPairs caused a Burst internal error when safety
    checks are enabled using Burst 1.6 or higher. A harmless workaround to the
    bug was discovered.

## [0.4.1] – 2021-9-16

Officially supports Entities [0.17.0]

### Fixed

-   Fixed `DistanceBetween()` queries for nearly touching box colliders where
    the edges were incorrectly reported as the closest points.
-   Fixed `ColliderCast()` queries for sphere vs box where negative local axes
    faces could not be hit.
-   Fixed `ColliderCast()` queries for sphere vs box where the wrong edge
    fraction was reported.
-   Fixed `ColliderCast()` queries for capsule vs capsule where a hit at the
    very end of the cast would report a zero distance hit.
-   Fixed `ColliderCast()` queries involving compound colliders incorrectly
    reporting a hit if the colliders start in an overlapping state.
-   Fixed `ColliderCast()` queries for capsule casters vs sphere targets where
    the wrong transforms were used.
-   Fixed `ColliderCast()` queries for capsule vs capsule where the query was
    executed in the wrong coordinate space.
-   Fixed `ColliderCast()` queries for box casters vs sphere targets where the
    start and end points were flipped, causing incorrect results to be
    generated.
-   Fixed argument names in `ColliderCast()` queries involving compound
    colliders.

### Improved

-   `ColliderCast()` queries for capsule vs box use a new more accurate
    algorithm.
-   `ColliderCast()` queries for box vs box use a new algorithm which is both
    faster and more accurate.

## [0.4.0] – 2021-8-9

Officially supports Entities [0.17.0]

### Added

-   Added `StepVelocityWithInput()`. It is the input-driven smooth motion logic
    I have been using for several projects.
-   Added several methods and utilities to aid in ballistics simulations. This
    API is subject to change as I continue to explore and develop this aspect of
    Psyshock. The new stuff can be found in
    *Physics/Dynamics/Physics.BallisticUtils.cs*.
-   *New Feature:* Added `Physics.ColliderCast()` queries.
-   *New Feature:* Added `DistanceBetween()` queries that take a single point as
    an input.
-   Added `subColliderIndex` to `PointDistanceResult` and `RaycastResult`.
-   Added `CombineAabb()`.

### Changed

-   **Breaking:** Renamed `CalculateAabb()` to `AabbFrom()`.

### Fixed

-   Fixed an issue where `DistanceBetween()` queries did not properly account
    for the center point of `SphereColliders`.

## [0.3.2] – 2021-5-25

Officially supports Entities [0.17.0]

### Added

-   Added constructor for `PhysicsScale` to auto-compute the state for a `float3
    scale`

### Fixed

-   Fixed compound collider scaling for AABBs and `DistanceBetween()`
-   Fixed `BuildCollisionLayer.ScheduleSingle()` using the wrong parallel job
    count
-   Fixed issue where a user could accidentally generate silent race-conditions
    when using `PhysicsComponentDataFromEntity` and `RunImmediate()` inside a
    parallel job. An error is thrown instead when safety checks are enabled.
-   Renamed some files so that Unity would not complain about them matching the
    names of `GameObject` components

## [0.3.0] – 2021-3-4

Officially supports Entities [0.17.0]

### Added

-   *New Feature:* Added support for Box Colliders
-   Added implicit conversion operators from `ComponentDataFromEntity` to
    `PhysicsComponentDataFromEntity` and from `BufferFromEntity` to
    `PhysicsBufferFromEntity`. You can now use `Get###FromEntity()` instead of
    `this.GetPhysics###FromEntity()`.
-   Added `subColliderIndexA` and `subColliderIndexB` to
    `ColliderDistanceResult`, which represent the indices of the collider hit in
    a `CompoundCollider`

### Changed

-   **Breaking:** `Latios.Physics` assembly was renamed to `Latios.Psyshock`
-   **Breaking:** The namespace `Latios.PhysicsEngine` was renamed to
    `Latios.Psyshock`
-   **Breaking:** `LatiosColliderConversionSystem` and `LatiosColliderAuthoring`
    have been renamed to `ColliderConversionSystem` and `ColliderAuthoring`
    respectively

### Fixed

-   Added missing `Conditional` attribute when building a collision layer from
    an `EntityQuery`

### Improved

-   All authoring components have now been categorized under *Latios-\>Physics
    (Psyshock)* in the *Add Component* menu

## [0.2.2] - 2021-1-24

Officially supports Entities [0.17.0]

### Added

-   *New Feature:* The FindPairs dispatcher from the upcoming [0.3.0] has been
    backported in order to better support 2020.2 users
-   `FindPairs.ScheduleParallel` now checks for entity aliasing when safety
    checks are enabled and throws when a conflict is discovered
-   `FindPairs.WithoutEntityAliasingChecks` disables entity aliasing checks
-   `FindPairs.ScheduleParallelUnsafe` breaks aliasing rules for increased
    parallelism
-   When safety checks are enabled, FindPairs bipartite mode validates that the
    two layers passed in are compatible

### Changed

-   `CompoundColliderBlob.colliders` is now a property instead of a field

### Fixed

-   FindPairs jobs now show up in the Burst Inspector and no longer require
    BurstPatcher
-   Fixed a bug when raycasting capsules
-   Properly declare dependencies during collider conversion
-   Fixed warning of empty tests assembly

## [0.2.0] - 2020-9-22

Officially supports Entities [0.14.0]

### Added

-   *New Feature:* Compound Colliders are here
-   *New Feature:* `BuildCollisionLayer` schedulers
-   Added `PhysicsComponentDataFromEntity` and `PhysicsBufferFromEntity` to help
    ensure correct usage of writing to components in an `IFindPairsProcessor`
-   Added `PatchQueryForBuildingCollisionLayer` extension method to
    `FluentQuery`
-   Added `Physics.TransformAabb` and `Physics.GetCenterExtents`

### Changed

-   Collider conversion has been almost completely rewritten
-   Renamed `AABB` to `Aabb`
-   `CalculateAabb` on colliders is now a static method of `Physics`
-   Renamed `worldBucketCountPerAxis` to `worldSubdivisionsPerAxis` in
    `CollisionLayerSettings`

### Removed

-   Removed `GetPointBySupportIndex`, `GetSupportPoint` and `GetSupportAabb`
    methods on colliders
-   Removed `ICollider`
-   Removed `CollisionLayerType` as it was unnecessary
-   Removed `Physics.DistanceBetween` for a point and a sphere collider as this
    API was not complete

### Fixed

-   Fixed issues with capsule collider conversion
-   Fixed issues with extracting transform data from entities in
    `BuildCollisionLayer`

### Improved

-   Many exceptions are now wrapped in a
    `[Conditional("ENABLE_UNITY_COLLECTION_CHECKS")]`
-   `Collider` now uses `StructLayout.Explicit` instead of `UnsafeUtility` for
    converting to and from specific collider types
-   `BuildCollisionLayer` using an `EntityQuery` no longer requires a
    `LocalToWorld` component

## [0.1.0] - 2019-12-21

### This is the first release of *Latios Physics for DOTS*.
