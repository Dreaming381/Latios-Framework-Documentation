# Calligraphics Cheat Sheet

## Authoring

-   `TextRendererAuthoring` – The *Text Renderer* component, all you need to
    render text
-   `FontCollectionAsset` – Editor-only asset telling Calligraphics where fonts
    live
    -   `ProcessFonts()` – What the *Process!* button calls
-   `FontCollectionAuthoring` – Bakes fonts without a *Text Renderer*
-   `TextGradientAuthoring` – Authors the named gradients rich text can call out
-   `FontUtility` – Inspector tool for reading face and variable axis metadata
-   `BakeLanguageStringBlob()` – `IBaker` extension for BCP 47 language blobs

## Text Content

-   `DynamicBuffer<CalliByte>` – Where the UTF-8 text itself lives
-   `CalliString` – Wraps that buffer in the Collections string APIs
    -   Constructs implicitly from a `DynamicBuffer<CalliByte>`
    -   `Length`, `Clear()`, and `TryResize()` – Basic mutation
    -   `Equals()` and `CompareTo()` – Against `string`, `NativeText`, and the
        `FixedString` sizes
-   `CalliString.Enumerator` – Walks the text as `Unicode.Rune` values
    -   `MoveNext()` and `MovePrevious()` – Steps either direction
    -   `GotoByteIndex()` – Constant-time jump to a byte offset

## Text Configuration

-   `TextBaseConfiguration` – Runtime counterpart of the authoring settings,
    freely writable
    -   `SetFamily()` – Sets the default font by family name
    -   `fontSize`, `color`, `maxLineWidth`, `wordWrap`, `isOrthographic`
    -   `wordSpacing`, `lineSpacing`, `paragraphSpacing` – Font units of 0.01 em
    -   `language` – A `LanguageBlob` hinting HarfBuzz toward a locale
-   `FontStyles` – Bold, italic, case conversion, super/subscript, fraction
-   `FontWeight` and `FontWidth` – Thin through Black, UltraCondensed through
    UltraExpanded
-   `HorizontalAlignmentOptions` and `VerticalAlignmentOptions` – Also the pivot
    the text expands from
-   `FontTextureSize` – Normal, Big, or Massive texture rasterization budget
-   `TextRendererUtility` – Runtime building APIs
    -   `BakeLanguage()` – A BCP 47 or OpenType tag into a `LanguageBlob`
    -   `GetTextBaseConfiguration()` – Every setting as one call

## Fonts

-   `FontLoadDescription` – One face within one font file
    -   `GetDescriptionsFromPath()` – Discovers the faces in a file at runtime
-   `FontLoadDescriptionsBlobReference` – Auto-loads fonts when attached to an
    entity

## Rendering and Animation

-   `DynamicBuffer<RenderGlyph>` – The generated quads, one per glyph
    -   `bl`, `br`, `tl`, `tr` prefixes – Position, UVB, and color per quad
        corner
-   `DynamicBuffer<AnimatedRenderGlyph>` – Add this to override the glyphs for
    animation
-   `CalligraphicsAnimationSuperSystem` – Inject animation systems here
-   `DynamicBuffer<TextColorGradient>` – Named gradients on the
    `worldBlackboardEntity`
-   `TextBackendBakingUtility` – Provides the shared mesh, for building text
    entities from scratch

## Bootstrap

-   `CalligraphicsBootstrap.InstallCalligraphics()` – Installs the module into a
    world
