# Changelog

All notable changes to this package will be documented in this file.

The format is based on [Keep a Changelog](http://keepachangelog.com/en/1.0.0/)
and this project adheres to [Semantic
Versioning](http://semver.org/spec/v2.0.0.html).

## [0.15.3] – 2026-5-2

Officially supports Entities [1.4.6]

### Fixed

-   Fixed build issue due to `UnityEditor` reference in `FontCollectionAsset`

## [0.15.2] – 2026-4-25

Officially supports Entities [1.4.6]

### Fixed

-   Fixed missing dependency in an `EntityQuery` for a job causing exceptions
    for some entity lifecycle patterns

## [0.15.1] – 2026-4-19

Officially supports Entities [1.4.4]

### Fixed

-   Fixed exceptions on shutdown when Calligraphics is installed but not used

## [0.15.0] – 2026-4-12

Officially supports Entities [1.4.4]

### Added

-   *New Feature:* Added support for diacritics, non-latin scripts, emoji, and
    other advanced text rendering features
-   *New Feature:* Added support for dynamic font atlases and loading fonts at
    runtime
-   Added support for individually animating each vertex corner of a glyph quad
    on the CPU
-   Added support for HDR vertex colors

### Changed

-   **Breaking:** The entire text pipeline has been reworked to no longer depend
    on Unity’s TextCore and instead rely on HarfBuzz directly, meaning
    everything involving management of fonts and materials has been altered to a
    new workflow featuring font-independent materials and single-entity text
    renderers
-   **Breaking:** RenderGlyph now uses a new 128-byte per glyph format
-   GPU Resident Text is now determined automatically based on usage patterns
-   Shaders are now distributed as samples rather than as part of the package
    (and there’s more shaders to choose from)

### Fixed

-   Fixed comparison exceptions for CalliString

### Improved

-   Animation no longer requires reprocessing the input UTF-8 text
-   The size of the text backend mesh on disk has been reduced significantly

### Removed

-   **Breaking:** Removed built-in text animation system and all associated
    APIs, although user animation can still be done within
    `CalligraphicsAnimationSuperSystem` using `AnimatedRenderGlyph`
-   **Breaking:** Removed `GlyphMapper`, `GlyphMappingElement`, and
    `GlyphMappingMask`, as these are incompatible with HarfBuzz shaping
-   **Breaking:** Removed `TextRenderControl`, as there is a completely new
    upload pipeline

## [0.14.9] – 2026-1-11

Officially supports Entities [1.4.4]

### Fixed

-   Fixed reflection errors for Unity 6.3 and newer

### Removed

-   Removed the *Text BackendMesh* from the *Asset/Create* menu

## [0.14.7] – 2025-12-21

Officially supports Entities [1.4.3]

### Fixed

-   Fixed LATIOS_DISABLE_CALLIGRAPHICS not being considered by the editor
    assembly

## [0.14.6] – 2025-12-20

Officially supports Entities [1.4.3]

This release only contains internal compatibility changes

## [0.14.5] – 2025-12-13

Officially supports Entities [1.3.14]

### Fixed

-   Fixed compilation for Unity 6.3 and newer

## [0.14.0] – 2025-10-18

Officially supports Entities [1.3.14]

This release only contains internal compatibility changes

## [0.13.5] – 2025-8-30

Officially supports Entities [1.3.14]

### Added

-   Added scripting define LATIOS_DISABLE_CALLIGRAPHICS which disables the
    module entirely, as Unity 6.3 alpha breaks the current implementation’s
    ability to compile

## [0.12.0] – 2025-2-23

Officially supports Entities [1.3.9]

### Changed

-   Altered the authoring UI to be compatible with Unity 6

### Improved

-   Changed authoring menu items to be more obvious they come from Calligraphics
-   Updated shaders to target Unity 6, which will hopefully decrease the chance
    of shader errors

## [0.11.0] – 2024-9-29

Officially supports Entities [1.3.2]

### Changed

-   Round-robin rendering dispatch systems are now Burst-compiled `ISystems`

## [0.10.6] – 2024-8-3

Officially supports Entities [1.2.1] – [1.2.3]

### Fixed

-   Fixed animation regression due to change filtering preventing resetting
    modified glyphs each frame

## [0.10.0] – 2024-5-26

Officially supports Entities [1.2.1]

### Added

-   *New Feature:* Added Text Renderer support for many new base text formatting
    options
-   *New Feature:* Added support for many more rich text tags
-   *New Feature:* Added support for multiple fonts in a single text renderer
-   *New Feature:* Added support for GPU-resident text, allowing text that
    rarely changes to be cached on the GPU
-   *New Feature:* The included shader and shader subgraph for text now supports
    bold highlights and other small quality improvements
-   Added `TextCore` `FontAsset` extension method
    `GetGlyphPairAdjustmentRecords()`
-   Added `CalliString.Enumerator` method `GotoByteIndex()` which allows moving
    the enumerator to a specific byte in constant time, although invalid offsets
    result in undefined behavior
-   Added `CalliString.Enumerator` properties `CurrentByteIndex` and
    `CurrentCharIndex`

### Changed

-   **Breaking:** The vertical alignment options for Text Renderers both in
    authoring and at runtime have changed to support anchoring to specific font
    lines in the text
-   **Breaking:** `AlignMode` at runtime has been replaced with separate
    `HorizontalAlignmentOptions` and `VerticalAlignmentOptions` which are
    internally packed into `TextBaseConfiguration`
-   **Breaking:** `FontBlob` has been heavily reworked to support the new
    functionality and improve performance

### Fixed

-   Fixed `TextRenderControl.ApplyShearToPositions` using the wrong shear angle
-   Fixed animation indexing when the index exceeds the amount remaining in the
    text so that the excess is ignored, whereas previously this resulted in
    undefined behavior for the entirety of the animation
-   Fixed kerning pairs

### Improved

-   Glyph Generation performance has significantly improved
-   Added change filters to glyph generation and rendering updates

## [0.9.0] – 2024-1-20

Officially supports Entities [1.1.0-pre.3]

### Added

-   Added `Rendering` namespace which contains the former `TextBackend`
    namespace from Kinemation as now Calligraphics owns the text rendering
    process

## [0.8.4] – 2023-11-10

Officially supports Entities [1.0.16]

### Fixed

-   Fixed URP shader alpha clipping issue introduced in a URP patch

## [0.8.0] – 2023-9-23

Officially supports Entities [1.0.16]

### This is the first release of *Calligraphics*.
