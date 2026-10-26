# Changelog

All notable changes to this package will be documented in this file.

The format is based on [Keep a Changelog](http://keepachangelog.com/en/1.0.0/)
and this project adheres to [Semantic
Versioning](http://semver.org/spec/v2.0.0.html).

## [0.16.4] – 2026-10-25

Officially supports Entities [1.4.8]

### Fixed

-   Fixed memory corruption issue when an entity loses all components
-   Fixed memory corruption issue when an `AuxQueryEnumerator<>` is iterating
    and a new component type is added to an entity that has never been seen
    before
-   Fixed `AuxWorld.AllOf<T>` corrupting memory when the specified type has
    never been added to an entity

## [0.16.2] – 2026-10-17

Officially supports Entities [1.4.8]

### Fixed

-   Fixed various memory management and corruption issues caused by component
    removal
-   Fixed `AuxComponentEnumerator<T>` accessing stale or invalid memory when a
    new component type is added to an entity that has never been seen before
    while the enumerator is active
-   Fixed `AuxWorld` method `RemoveAllComponents()` not actually doing anything

## [0.16.0] – 2026-9-20

Officially supports Entities [1.4.8]

### Improved

-   Added UNITY_DOTS_DEBUG support to safety checks

### Removed

-   Removed `TestDisposable` since it has served its purpose

## [0.15.4] – 2026-5-16

Officially supports Entities [1.4.6]

### Fixed

-   Fixed exceptions when performing certain queries
-   Fixed queries not returning matching entities
-   Fixed exceptions when removing a component from an entity
-   Fixed exceptions when disposing an `AuxWorld`
-   Fixed memory leaks
-   Fixed `AuxWorld` not being initialized correctly

## [0.15.0] – 2026-4-12

Officially supports Entities [1.4.4]

### This is the first release of *Aux ECS*.
