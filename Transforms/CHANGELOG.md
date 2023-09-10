# Changelog

All notable changes to this package will be documented in this file.

The format is based on [Keep a Changelog](http://keepachangelog.com/en/1.0.0/)
and this project adheres to [Semantic
Versioning](http://semver.org/spec/v2.0.0.html).

## [0.8.0] – 2023-9-?

Officially supports Entities [1.0.14]

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
