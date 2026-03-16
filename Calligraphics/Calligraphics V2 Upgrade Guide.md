# Calligraphics V2 Upgrade Guide

Calligraphics V2 is a major overhaul of the text rendering pipeline. The effort
was started by Fribur in the [TextMeshDOTS
fork](https://github.com/Fribur/TextMeshDOTS), and further collaborative efforts
have evolved it to where it is today.

Upgrading unfortunately requires a reauthoring of assets, and perhaps some
runtime changes as well. But hopefully the new features make these changes
worthwhile!

## Paradigm Shift

Calligraphics was originally written to target Unity’s TextCore engine and
static fonts. However, TextCore proved to be insufficient for advanced text
rendering features, especially dynamic fonts.

Calligraphics V2 moves away from engine dependencies, and instead comes packaged
with its own version of [HarfBuzz](https://github.com/harfbuzz/harfbuzz) for
processing text. HarfBuzz adds many features including diacritics and emojis,
but there’s also some old features that are incompatible with HarfBuzz and have
been removed.

The other big change is that materials are now font-independent. This means that
a text string can contain multiple fonts and be rendered by a single material.
Calligraphics V2 achieves this using runtime global texture arrays as large
atlases of signed-distance fields and bitmaps for emojis. This new approach is
much more powerful, but it also requires a new authoring workflow.

## Authoring Fonts

Before starting an upgrade, make sure to create a backup of your project. You
may want to open up that backup in a separate editor instance so that you can
reference old settings.

Calligraphics V2 works with .ttf, .otf, and .ttc fonts directly. You may delete
any TextCore font assets.

Fonts can either be streaming assets, or system fonts. System fonts are fonts
that are pre-installed on the target device which your application loads from
disk. Whereas streaming asset font are fonts you bundle with your application.
For each system font you want to reference, you must have a copy of it somewhere
in your project. Streaming asset fonts must be in a *StreamingAssets* folder.

Next, you must create a *Font Collection Asset*.
(*Assets-\>Create-\>Latios-\>Calligraphics-\>Font Collection Asset*) Assign font
references to the *System Fonts* and *Streaming Asset Fonts* lists, and then
click the *Process!* button. This should populate the lists below the button.

If you plan to set up text entities fully at runtime, you may want to add a
*Font Collection* authoring component to some Game Object in a subscene.

Typically, you will only have one Font Collection Asset in your project, but you
can have multiple. At runtime, once Calligraphics loads a font, it does not
unload it until shutdown, so there is little benefit to splitting fonts into
multiple collections for streaming purposes.

## Authoring Text Renderers

Calligraphics V2 still uses the *Calligraphics/Text Renderer* authoring
component to configure text. Most of the settings are the same as before.
However, there are a few key differences.

First, there’s a field to drop in the Font Collection Asset. Once you do this,
the *Default Font* dropdown should allow you to select a font to render the text
with.

Calligraphics V2 no longer supports disabling kerning. Kerning is always
enabled.

New to Calligraphics V2 is the language selector. This is an IETF BCP 47
language tag which can be used to help format the text with the correct rules.

Also new is the *Font Texture Size*. Use *Normal* for typical text use cases.
Use *Big* for large title text. Only use *Massive* when *Big* is visibly
insufficient, as it is extremely resource-intensive and completely overkill for
nearly all use cases.

Calligraphics V2 only allows a single material per text renderer. However, a
single material can express much more than before. You will need a compatible
shader. Unlike in V1, V2s shaders come as package samples which you can install,
copy, and modify as you see fit.

Lastly, there’s no *GPU Resident* option anymore. This is now determined
automatically at runtime.

## Writing to Components

The `TextRendererAspect` type has been removed. Instead, you are encouraged to
construct a `CalliString` local variable directly from the
`DynamicBuffer<CalliByte>` whenever you need to modify the text string. For all
other properties, you can modify the `TextBaseConfiguration` component directly.

Perform all text modifications prior to
`Unity.Rendering.UpdatePresentationSystemGroup`.

## Animating Text

Due to various changes and the introduction of HarfBuzz, the previously included
text animation system has been removed. However, it is still possible to animate
text.

To do so, add the `AnimatedRenderGlyph` dynamic buffer to your text renderer
entities. In a system, you can read from the `RenderGlyph` dynamic buffer to
obtain the generated glyph data. This data will only change whenever the input
text and configuration changes, so you can use change filters for it. Your
system is responsible for updating the `AnimatedRenderGlyph` dynamic buffer with
a modified (animated) copy of the `RenderGlyph` buffer. Your system must update
inside `CalligraphicsAnimationSuperSystem`.

## Other Noteworthy APIs and Features

Calligraphics V2 has a few new tools and features to take advantage of.

### FontUtility

`FontUtility` is a scriptable object you can create as an asset and use as a
tool to reveal font metadata.

### FontLoadDescription

If necessary, you can create a `BlobAssetReference<FontLoadDescriptionBlob>` and
assign it to a `FontLoadDescriptionBlobReference` component attached to any
entity. This allows you to load fonts dynamically at runtime, including from
file paths that aren’t system fonts nor Streaming Asset fonts.

You can safely dispose the blob asset after a full frame has passed.

## Missing Something?

The current feature set of Calligraphics is the work of several contributors. If
you find Calligraphics is missing functionality you desire and would like to
help add that functionality, feel free to reach out on the framework discord!
Additionally, if you are having trouble meeting performance targets with the new
solution, please report these challenges!
