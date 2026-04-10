# Calligraphics

Calligraphics is a module responsible for rendering world-space UI elements
(primarily text) in pure ECS fashion, largely inspired by TextMesh Pro. It
combines Kinemation’s rendering utilities, the HarfBuzz library, and a custom
signed-distance-field implementation to generate and render beautiful
world-space text.

This module has a strong collaborative history, first started by Sovogal, it was
brought closer to TextMesh Pro parity by Fribur. Fribur then went on to create
the [TextMeshDOTS](https://github.com/Fribur/TextMeshDOTS) fork, developed the
HarfBuzz integration, and then further collaboration has evolved both packages
to where they are today.

Check out the [Getting Started](Getting%20Started.md) page!

## Features

### Out-of-the-Box

Calligraphics does not require any user code in order to render text in your
world. Everything can be configured in the editor, with a workflow that is
intuitive and just works.

### Advanced Text

Calligraphics supports various text features, such as kerning, diacritics, and
even emoji in various formats, such as PNG, COLR, SVG, and BMP. With HarfBuzz it
will automatically attempt to format text based on the detected writing system,
and allows you to further hint it with language tags.

### Plenty of Formatting and Layouting

Many of the essential formatting options are available, including bold, italics,
case conversion, superscripts and subscripts, and even fraction rendering. Even
more formatting can be unlocked through rich text tags.

In addition, Calligraphics supports various text alignment options as well as
customizations in text size and resource use, so you can be sure text is crisp
at whatever size and place you need.

### Any Font Any Time

Calligraphics is designed to decouple rendering from the fonts used. Fonts can
be included from the build, loaded from the operating system, or even loaded
from downloaded files or custom paths at runtime. Any Calligraphics-compatible
material can render any font, or any combination of fonts, thanks to a global
texture atlas array mechanism that all Calligraphics shaders can tap into.

Calligraphics comes with several shader-graph shaders with supports for various
outlines and effects, with support for both URP and HDRP.

### Fast SDF Rendering

Calligraphics dynamically generates signed-distance field textures of new
characters at runtime using a custom SIMD-optimized rasterizer. Rendering text
from SDFs is very efficient on modern GPUs, including mobile GPUs. SDF texture
data is uploaded sparsely with a custom sparse texture uploader that outperforms
Unity’s texture upload process.

### Animatable

Calligraphics provides an API that allows for animating the generated glyph
quads it has generated from the text before sending them to the GPU. The system
automatically recognizes which text is being animated and which isn’t, and
optimizes GPU allocations appropriately to account for this.

## Known Issues

-   RTL, BiDi, and advanced unicode line-breaking are not yet supported.
-   There is no way to map output glyphs to input UTF-8 text for animation
    currently.
-   Some advanced effects such as highlighting, underlines, and strikethrough do
    not work.

## Potential Future Roadmap Items

-   More text formatting and layouting features
-   Custom Sprites
-   Culling Optimizations
-   Voxel per glyph mode (for raymarched text)
-   Billboards
-   9-Slice Panels
-   Decals
