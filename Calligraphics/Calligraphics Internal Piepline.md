# Calligraphics Internal Pipeline

Calligraphics has an intricate text pipeline that has evolved over time through
various collaborations. This guide documents the current pipeline and highlights
various shortcomings and potential areas of future improvements.

You do not need to read this guide in order to use Calligraphics. However, if
you wish to help contribute to its features or are debugging an issue you
encountered, then this guide may help provide some insight to get started.

## Tables

Caligraphics defines various persistent structures referred to as *tables*, each
stored in a Collection Component.

### Font Table

The `FontTable` stores font information about all loaded fonts. It contains a
list of HarfBuzz Face objects, all marked as immutable so they can be read from
any thread. The `FontTable` also has a per-thread HarfBuzz Font object, which is
lazily created on each thread when needed. It also contains a couple of hashmaps
that map `FontLookupKey` instances to `Face` objects and named variations for
variable fonts. `FontLookupKey` is often constructed at runtime from the
combination of `TextBaseConfiguration` and rich text tags.

Most of the `FontTable` is constructed and populated by
`NativeFontLoaderSystem`.

### Glyph Table

The `GlyphTable` is responsible for storing unique metadata that describes a
particular glyph shape within a font. The table is populated dynamically at
runtime as new text is encountered requiring new glyphs from fonts. Once a glyph
is added to the table, it is never unloaded. The `GlyphTable` is populated with
`Entry` instances. Each `Entry` is composed of a `Key`, and atlas information.

The `Key` is what uniquely identifies a glyph within the `GlyphTable`. It is
composed of a `Face` index within the `FontTable`, a glyph index stored within
the HarfBuzz `Face` object, what type of texture data it uses, how big it is in
the texture, and a variation index. Currently, the variation index refers to a
named variation within the `Face` object.

The atlas information describes the size of the glyph in pixels both with and
without padding, as well as padding offsets. This information is used to
generate the `RenderGlyph` quad correctly. A glyph may or may not be in an
atlas. If it is, its atlas coordinates are also stored in the `Entry`. The
`Entry` also contains a count of how many `RenderGlyph` and
`AnimatedRenderGlyph` instances across the ECS World use the glyph.

While the `GlyphTable` is typically indexed via a `Key` using a hashmap, it can
also be indexed by the index of an `Entry` directly. These indices are stored in
`RenderGlyph` instances.

### Glyph GPU Table

The `GlyphGpuTable` is responsible for managing free space available in a
`ByteAddressBuffer` containing all the glyph quad information. Each glyph quad
is a `RenderGlyph`, consuming 128 bytes of data, and is unpacked directly in the
vertex stage of the shader. Each renderer has a `TextShaderIndex` component
which specifies a range of glyphs in the buffer that represent the text it must
render.

### Atlas Table

The `AtlasTable` is responsible for managing allocations of glyph texture data
stored in the GPU. Each glyph is stored in one of three different
`RenderTexture` objects, with each `RenderTexture` being an array of slices.
Each slice is broken up into shelves, where a shelf represents a height range in
pixels that is a multiple of 16. Each glyph’s height is rounded up to the
nearest 16, and then it is allocated in a shelf of that height. Each shelf
stores a list of gaps along the x-axis that can be filled.

Unlike many other resources, glyphs can be deallocated from the `AtlasTable` at
runtime to make room for other glyphs. However, shelves and slices stick around
once allocated. Adding support for shelf removal might help with atlas
fragmentation in the future.

## Pipeline Overview

Calligraphics’ text pipeline can be broken down into four stages:

-   Glyph Generation
-   Glyph Animation
-   GPU Dispatch
-   Shaders

Glyph Generation is responsible for converting the input UTF-8 strings and
`TextBaseConfiguration` into a sequence of `RenderGlyph` instances. It makes
heavy use of `HarfBuzz`.

Glyph Animation is entirely user-defined within the
`CalligraphicsAnimationSuperSystem`, and will not be touched on further.

GPU Dispatch is the process of updating the text renderers, generating texture
data, and uploading everything to the GPU.

Lastly, the shaders need to decode all the text data and render the text, often
using SDF algorithms.

## Glyph Generation

Glyph generation is the process of converting the text and its formatting
options into a sequence of `RenderGlyph`s that can be used for rendering. The
entire process is handled by `GenerateGlyphsSystem`. But this is not a small
system, and the process can be broken down into several discrete steps.

The first step is to pre-allocate the `DynamicBuffer<RenderGlyph>` of each
entity with a changed `DynamicBuffer<CalliByte>` on a single thread. We allocate
a full `RenderGlyph` per UTF-8 byte. We’ve found doing this single-threaded
speculatively in advance is faster than allocating the glyphs when we actually
generate them due to how Unity’s Persistent allocator works. This may change in
\~Unity 6.5 when Unity switches to mimalloc.

After that, we extract rich text XML tags and write out their metadata to a
`NativeStream`.

After that, we run a shaping job. Shaping is the process where we give text
spans to HarfBuzz with various metadata, and ask it to give us the glyph IDs and
vertex positions of the quads for us to place the glyphs. This job gets a little
complicated, so let’s break it down further.

For each text entity, we load in all the rich text tags. If there are no rich
text tags, we take a fast-path and do all the shaping in a single shot.
Otherwise, we break up the string into smaller spans identifying each unique
region between rich text tags so that we can shape each span.

Our `Shape()` method picks out the HarfBuzz font object, configures it with our
settings, and then calls into HarfBuzz’s shaping function. After that, we fetch
the arrays of `GlyphInfo` and `GlyphPosition` from HarfBuzz. We iterate those
arrays to generate a `GlyphOTF` structure, composed of a `GlyphTable.Key`, the
cluster, and the glyph offset info. We write that struct to a `NativeStream`.

We also check if the `GlyphTable.Key` is present in the `GlyphTable`. If not, we
then try to add it to a per-thread hashset. If we succeed, then we write the
`Key` into another `NativeStream` of missing `Key` instances. Glyphs tend to
repeat a lot, so the per-thread hashset deduplicates new glyphs a bit, which
helps reduce the work of the next single-threaded job.

It is worth noting at this point that HarfBuzz has laid out all the text on a
single, potentially very long line. This would normally be the point where we
would handle line breaking, RTL, BiDi, glyph \<-\> char mapping for animation
logic, and all those things. But due to how Calligraphics evolved over time, we
do line breaking later on. This is certainly an area that can be improved!

`AllocateNewGlyphsJob()` is a short single-threaded job where we aggregate all
the new glyphs from all the worker threads, deduplicate them, allocate
`GlyphTable.Entry` elements for them, and add their `Key`s to the hashmap that
looks up the entries.

`PopulateNewGlyphsJob()` is where we actually populate the `GlyphTable.Entry`
elements. You might think this job is trivial, but it is actually quite
expensive. This job needs to compute the extents of the glyph. And in order to
do that, it needs to analyze the contours. Or in the case of an emoji, it needs
to analyze all the paint commands, which there can be hundreds of. We wanted to
parallelize this job, but unfortunately, the call into HarfBuzz to actually
obtain these extents has a bit of a flaw. HarfBuzz locks a mutex to obtain a
global scratch buffer it uses to parse the glyphs. Running this job in parallel
actually slows everything down. At some point, I’d like to see a contribution to
HarfBuzz to make these scratch buffers `thread_local` so that they don’t need to
be mutex-protected.

With the `GlyphTable` populated, we can now run the `GenerateGlyphsJob` and
construct the `RenderGlyph`s using everything we’ve generated so far. In this
job, we calculate the vertices, vertex colors, and other pieces of metadata. It
is also here that we perform line-breaking and line layouting, which is not
ideal.

After that, we have all our glyphs generated. The next step is to dispatch them
to the GPU.

## GPU Dispatch

Now we know the glyphs we want to render. It is time to put them into GPU
buffers. But before we do that, we need to figure out what all changed, so that
we can track resources appropriately. We do this in
`UpdateGlyphRenderersSystem`.

### UpdateGlyphRenderersSystem

`UpdateGlyphRenderersSystem` is responsible for examining changed glyph buffers
and maintaining correct ref counts of each glyph. It also computes the bounds of
each text renderer. And it alters the submesh index of the `MaterialMeshInfo` to
select the number of quads to draw.

The system maintains ref counts incrementally by maintaining a
`DynamicBuffer<PreviousRenderGlyph>` which is a cleanup buffer. This buffer is
added during `TextRendererInitializeSystem`. That system also captures new
entities, which is used to trigger change filters on the `RenderGlyph` buffers
for further processing.

Text renderers go through a state machine which defines how their glyphs are
allocated on the GPU. This is handled through the `GpuState` component. The
component is enabled if glyphs require uploading, and disabled if they don’t.
Additionally, this component contains state which describes whether the glyphs
were previously allocated in GPU memory, and whether or not they changed
recently. Initially, an entity starts out in the `Uncommitted` state. When it is
first uploaded to the GPU, it is moved to the `Dynamic` state. If on the next
frame the glyphs change, it is demoted back to the `Uncommitted` state.
Otherwise, it is promoted to the `DynamicPromoteToResident` state. In the latter
case, once the glyphs are uploaded, the entity is promoted to the `Resident`
state. In the Resident state, the glyphs live in a part of the GPU buffer that
is expected to not change every frame. In future frames where the glyphs don’t
change, they won’t be reuploaded. The `ResidentRange` cleanup component stores
where in the buffer these glyphs are stored, so that they can be deallocated
when the entity is destroyed. Non-resident entities have glyphs uploaded to a
part of GPU memory that is overwritten every frame. This mitigates fragmentation
for animated text. Text that is resident, but then changes will either be
demoted back to Uncommitted, or if the number of glyphs is the same, will be
demoted to `ResidentUncommitted`. The latter allows the new glyph data to
overwrite the already-allocated region. The `UpdateGlyphRenderersSystem` is
responsible for all state transitions, except for transitions to `Dynamic` and
`Resident` states, as those are triggered by actual upload operations.

When `UpdateGlyphRenderersSystem` is updating ref counts, it does so by reading
the `glyphEntryId` field of `RenderGlyph`. The top two bits of this value encode
texture format and size information used by the shaders. The remaining bits are
an index into the `GlyphTable`. `UpdateGlyphRenderersSystem` collects all glyph
reference count changes, and then identifies all glyphs that have an active
atlas location but their ref count drops to 0. Rather than immediately remove
this glyph from the atlas, the system adds it to
`AtlasTable.atlasRemovalCandidates`. Similarly, if a ref count rises above 0,
the system tries to remove the glyph from `AtlasTable.atlasRemovalCandidates`.
We’ll discuss why we do this shortly.

### DispatchGlyphsSystem

Now that we’ve updated all the renderers, the final system is
`DispatchGlyphsSystem`. This system leverages Kinemation’s
CullingComputeDispatch loop, meaning it updates either after culling or during
the Custom Graphics pass. This system is responsible for two things. It uploads
glyphs, and it generates and uploads texture data.

This system starts by iterating entities that request glyphs to be uploaded. It
captures pointers to the `PreviousRenderGlyph` buffer, as well as pointers to
the `TextShaderIndex` and `ResidentRange` components, and stores them in a
`NativeStream`. It also scans for glyphs not currently in an atlas and adds them
to a `NativeParallelHashSet`.

Next, two single-threaded jobs run side-by-side.

The first job allocates the `RenderGlyph` ranges in the GPU `ByteAddressBuffer`
and sets the `TextShaderIndex` and `ResidentRange` components. It also
aggregates the captures into a list, and prunes empty ranges.

The second job is responsible for allocating atlas space for the glyphs not
currently in the atlas. There are three atlases. One is an 8-bit SDF texture,
the second is a 16-bit SDF texture used for Big and Massive text, and the third
is a 32-bit color texture for emoji. The appropriate atlas is picked, and the
job first tries to allocate the glyph into an existing slice. If that fails,
then the atlas removes all the `atlasRemovalCandidates` to free up space. Then
it allocates the glyph, potentially creating a new slice if necessary. This
garbage-collection strategy helps keep frequently-used glyphs in the atlas, even
if there are some frames where their reference counts drop to 0. There might
still be further improvements to this strategy, but it has proven effective
enough for now. The second job also allocates byte ranges for a pixel upload
buffer.

This concludes the Collect phase. In the Write phase, there are two independent
parallel jobs which run side-by-side.

The more intense of these jobs is `RasterizeJob`. This job is responsible for
generating the texture data for each glyph newly-added to the atlas. The job
uses an `atomicPrioritizer` to reorder the job indices so that the emojis are
rasterized first, since they are much more expensive. For emoji rasterization,
we call into HarfBuzz’s rendering functionality. This functionality was recently
introduced, and prior to that, TextMeshDOTS and developmental Calligraphics used
a custom emoji renderer. However, HarfBuzz’s implementation is faster and
supports more emoji format. That old code may be removed in the future.

For SDFs, we use a custom algorithm that works in four steps. First, the glyph
contours are subdivided into small linear segments. Currently, these are
subdivided to be less than a third of a pixel of error. However, there are
ongoing discussions about whether we should subdivide further for higher
quality, at the expense of more runtime processing. After we have all of our
linear segments, we use Calci to remove overlaps, so that all segments
exclusively define the real edges of the glyph. We need to do this because some
fonts are poorly-designed and create overlaps. After that, we rasterize the
glyph to determine which pixels are inside and outside the glyph, and we also
compute the distance of each pixel to its nearest edge. After that, we combine
these results to compute a signed distance value, and write that to the pixel
upload buffer.

There’s actually a little more going on than that. But to explain it, I need to
introduce the concept of **spread**. Spread is the number of pixels away from
the edge (either inside or outside) before the SDF values saturate. Larger
spread means more padding pixels around the glyph, more pixels to evaluate per
edge segment, and support for thicker outlines and bolder bolds. It is a
tradeoff. Calligraphics chooses a spread of 1/8th the reference font size used
for rendering the SDFs. For normal text, that is a font size of 96, so the
spread is 12. Notice how I mentioned this affects pixels to evaluate per edge?
Rather than loop through each edge per pixel, we loop through a range of pixels
per edge based on the bounding box of the edge inflated by the spread. This
drastically reduces the number of distance evaluations. On top of that, the
distance evaluations make use of Burst’s LLVM autovectorizer, so this is a very
competitive SDF rasterizer!

For the other Write job, the DispatchGlyphsSystem writes out the `RenderGlyphs`
to a GPU upload buffer. During this process, it patches the texture coordinates
with the atlas coordinates.

Finally, in the Dispatch stage, the `RenderGlyphs` are copied to a persistent
`ByteAddressBuffer` via compute shader, converting a `uint` atlas slice index
into a float in the process. The pixels are also uploaded via compute shader and
copied into the `RenderTextures`. This compute shader is a little more involved,
as it needs to account for 2D coordinate offsets and convert pixels into
floating point values. But after that, all the data required is in the GPU. The
last step is the shaders.

One thing to note is that `DispatchGlyphsSystem` does not currently account for
culling of the text renderers to skip uploads. The state machine is designed
with this use case in mind, but lots of regression testing would be required to
verify it works correctly.

## Shaders

The vertex shader of text uses the vertex ID and the `TextShaderIndex` to look
up the `RenderGlyph` and extract it. The per-vertex data is assigned to the
vertex, including custom interpolators for colors and UVs, and the rest is
packed into a third interpolator. The fragment shader then reads these
interpolator values and unpacks the constant data. The top 2 bits of the
`glyphEntryId` are used to determine which texture atlas to sample, and if it is
an SDF, what spread value to use. The spread value is important, because the
shaders need to combine the texels per dilation and the texel changes per
screen-space pixel to figure out partial pixel coverage correctly.

For emoji, the shader samples the color atlas with mipmapping turned on, and
skips all the SDF processing and layering.

The actual shader graphs in the samples make heavy use of Custom Function Nodes.
These functions are defined in Calligraphics/ShaderLibrary along with many other
utility functions if you want to write your own shaders from scratch.
