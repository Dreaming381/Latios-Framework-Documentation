# Changelog

All notable changes to this package will be documented in this file.

The format is based on [Keep a Changelog](http://keepachangelog.com/en/1.0.0/)
and this project adheres to [Semantic
Versioning](http://semver.org/spec/v2.0.0.html).

## [0.10.0] – 2024-5-26

Officially supports Entities [1.2.1]

This release only contains internal compatibility changes

## [0.9.2] – 2024-2-4

Officially supports Entities [1.1.0-pre.3]

### Fixed

-   Fixed incorrect evaluation for Freeform Directional blend trees

## [0.9.0] – 2024-1-20

Officially supports Entities [1.1.0-pre.3]

### This is the first release of *Mimic*.

The below changes represent the delta between Mecanim in Mimic compared to
Mecanim in Kinemation from 0.8.6.

### Added

-   *New Feature:* Added experimental support for Blend Shape animation when
    LATIOS_MECANIM_EXPERIMENTAL_BLENDSHAPES is defined

### Changed

-   **Breaking:** The namespace for Mecanim is now `Latios.Mimic.Addons.Mecanim`
-   **Breaking:** Mecanim is now installed with the method
    `InstallMecanimAddon()` in both `MimicBootstrap` and `MimicBakingBootstrap`
-   `MecanimController` is now an `IEnableableComponent`

### Fixed

-   Child motions are now baked with their correct names
-   Fixed weighting of clip blends when some clips are culled due to having
    negligible visual impact
-   Fixed animation time values during transitions
-   Fixed multiple bugs with root motion
-   Fixed runtime safety errors with optimized skeletons

### Removed

-   **Breaking:** Removed `MecanimControllerEnabledFlag`
