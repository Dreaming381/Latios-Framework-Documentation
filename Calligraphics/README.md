# Calligraphics

*Written by Sovogal – Edited by DreamingImLatios*

Calligraphics is a small module that leverages the Kinemation backend rendering
systems, Unity’s Shader Graph, and a little bit of pixie dust to render
world-space text in a similar manner to TextMesh Pro.

Check out the [Getting Started](Getting%20Started.md) page!

## Features

### Out-of-the-Box

Calligraphics can parse Unity-supported [Rich
Text](https://docs.unity3d.com/Packages/com.unity.ugui@1.0/manual/StyledText.html)
tags with rendering support for a subset of those. What Calligraphics supports,
it will render, and what it does not, it will ignore, so you can port existing
Rich Text without concern that you'll expose any unseemly alligator brackets to
your players.

### Solid-State

Unless you're animating text, Calligraphics is essentially idle after it has
parsed and rendered once.

### Compatible with any Signed Distance Field (SDF) Font

Just like TextMesh Pro, Calligraphics supports any SDF Font via a
custom-included shader. Custom shader graph nodes, subgraphs, and SDF utilities
are provided if you need more control.

### Entities Graphics Pipeline

Calligraphics leverages Kinemation’s rendering backend, and supports Entities
Graphics material property workflows. It uses batching and instancing for all
text at once, so you know it's fast!

### Rich Text Support

Calligraphics supports the exclusion of all recognized Rich Text tags from the
rendering engine, both start and optional end tags. It also has rendering
support for a small subset of tags:

-   Alpha
-   Color
-   Gradient
-   Size

Eventually, I plan to support the same set of tags as Unity TextCore, but these
can get you started.

### Animation

Calligraphics provides some basic animation tools to give your world space text
a little pep. Animations are independent of the main rendering system, so they
produce overhead only if you want to use them. Calligraphics currently supports
five types of animation:

-   Color
-   Position
-   Rotation
-   Opacity
-   Scale

## Known Issues

-   Limited tag support. Many fundamental tags, such as bold and italics, don't
    affect text rendering. I recognize this as a major issue and will be
    prioritizing tag support in future releases.
-   Animation systems require glyphs to regenerate each frame in order for
    transitions to work, even when the transition is complete. This performance
    overhead is not ideal.
-   Limited authoring support. Unity native support for Rich Text editing keeps
    finding itself on the back-burner. A rich text editor would offer the most
    comfortable authoring experience, but creating one from scratch is a TON of
    work.
-   The Calligraphics shader does not render text edges as sharply as it should.
    The cause of this is unknown. The shader is based on TextMeshPro
    4.0.0-pre.2.
-   While Calligraphics represents strings internally as UTF-8, it does not
    handle glyph generation mechanisms for all languages correctly. Feel free to
    get involved to help support your native language.

## Near-Term Roadmap

-   Commonly-used Tag support
-   Billboards
-   9-Slice Panels
-   Decals

## Not-So-Near-Term Roadmap

-   Animation overhaul (solid-state after transition is complete)
-   Complete tag support
