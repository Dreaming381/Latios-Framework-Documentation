# Changelog

All notable changes to this package will be documented in this file.

The format is based on [Keep a Changelog](http://keepachangelog.com/en/1.0.0/)
and this project adheres to [Semantic
Versioning](http://semver.org/spec/v2.0.0.html).

## [0.12.0] – 2025-2-?

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
