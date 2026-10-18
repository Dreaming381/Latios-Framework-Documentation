# Changelog

All notable changes to this package will be documented in this file.

The format is based on [Keep a Changelog](http://keepachangelog.com/en/1.0.0/)
and this project adheres to [Semantic
Versioning](http://semver.org/spec/v2.0.0.html).

## [0.16.2] – 2026-10-17

Officially supports Entities [1.4.8]

### Added

-   Added `UnikaBootstrap` method `InstallUnikaLiveBakingDeserialization()`
    which should be called when using Unika but not using its built-in
    serialization system that is installed with
    `InstallUnikaEntitySerialization()`

### Changed

-   The Scripts tab in the runtime inspector has been replaced with an inline
    inspector within the components tab, which while slightly less elegant is
    much more robust and reliable

### Fixed

-   Fixed live baking of scripts not correctly deserializing `Entity` and
    `BlobAssetReference` elements within scripts upon world patching
-   Fixed live baking not correctly updating a scripts buffer when all scripts
    are removed
-   Fixed various issues with the `InterfaceAuthoring` `PropertyDrawer`
-   Fixed several script indexing issues in `UnikaMultiScriptAuthoring`
-   Fixed issue with adding the first script to an entity after previously
    removing all scripts from the entity
-   Fixed issue with removing a script not updating the bloom masks correctly
-   Fixed `object` `Equals()` across all types
-   Fixed entity that used to have scripts before they were all removed
    reporting `isEmpty` as false
-   Fixed `Cached<T>` not actually caching anything
-   Fixed handling of `default`-initialized scripts in comparison and casting
    operations

## [0.16.0] – 2026-9-20

Officially supports Entities [1.4.8]

### Added

-   *New Feature:* Added `UnikaScriptAutoAuthoring<T>` which an authoring type
    can inherit to have authoring be auto-generated for a script
-   *New Feature:* Added a new *Scripts* tab to the Entity runtime inspector
    which shows the state of all scripts attached to the selected entity
-   *New Feature:* Added attributes `[Needs]` `[CopyNeeds]` and
    `[ComponentBlacklist]` which resolve to the `ComponentBrokerBuilder`
    extension method `WithContext<T>()` allowing scripts to prescribe the
    component accesses they need to a `ComponentBroker` within a project
-   Added `InterfaceAuthoring<T>` which serializes any authoring component for a
    script implementing the specified interface
-   Added `managedView` property to `AllScriptsCollection`, which provides a
    boxed view of the scripts useful for debuggers and inspectors
-   Added `HierarchyScriptResolver` for QVVS Transforms mode which can resolve
    scripts using a `TransformsKey`
-   Added `ScriptTypeExtraction` static method `GetScriptType()` which returns
    the system type of a type-erased `Script`

### Changed

-   **Breaking:** `ICachedScriptResolver<T>` property `baseResolver` was
    replaced with an extension method `GetBaseResolver()` which can return the
    base resolver by ref
-   `CompareTo()` methods for `Script`, `ScriptRef`, and related types are no
    longer deterministic, and may change run-to-run

### Improved

-   Added UNITY_DOTS_DEBUG support to safety checks
-   Improved runtime initialization performance by using a more optimal assembly
    qualifier
-   Added C\# Roslyn Analyzers for detecting common source generator mistakes
    and offering fixes
-   Added inspector properties for `ScriptRef` and related types

## [0.15.10] – 2026-7-11

Officially supports Entities [1.4.7]

### Fixed

-   Fixed codegen issue for interfaces that have multiple properties or indexers

## [0.15.5] – 2026-5-23

Officially supports Entities [1.4.6]

### Fixed

-   Fixed source-generated code missing a double-underscore prefix for some
    interface types

## [0.15.0] – 2026-4-12

Officially supports Entities [1.4.4]

### Fixed

-   Fixed source generators generating local variable names that conflicted with
    user-defined interface parameter names

## [0.14.6] – 2025-12-20

Officially supports Entities [1.4.3]

This release only contains internal compatibility changes

## [0.14.1] – 2025-11-2

Officially supports Entities [1.3.14]

### Fixed

-   Fixed C\#10 file scope namespaces not being handled correctly in source
    generators

## [0.14.0] – 2025-10-18

Officially supports Entities [1.3.14]

This release only contains internal compatibility changes

## [0.12.5] – 2025-4-12

Officially supports Entities [1.3.9]

### Fixed

-   Fixed `ScriptRef` resolving on buffers that had scripts removed such that
    the number of scripts in the buffer exactly equals what used to be index of
    the script referenced by `ScriptRef`

## [0.12.4] – 2025-4-5

Officially supports Entities [1.3.9]

### Added

-   Added `DynamicBuffer<UnikaScripts>` extension method `CopyScriptFrom()`

## [0.12.0] – 2025-2-23

Officially supports Entities [1.3.9]

### This is the first release of *Unika*.
