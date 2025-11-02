# ![](media/60f09d8726e0a13095c19b993f2fb88f.gif)

# Kinemation Animation and Rendering

Kinemation is an animation and rendering solution for Entities designed to
service the performance needs of highly dynamic and complex worlds. Kinemation
replaces key parts of the Entities Graphics package to add additional features
and dramatically improve performance. It also provides the essential building
blocks for character animation.

Check out the [Getting Started](Getting%20Started%20-%20Part%201.md) page!

## Features

### Shader Compatibility and Familiar Workflows

Kinemation offers full shader compatible for all shaders compatible with
Entities Graphics and GPU Resident Drawer. This is true for both QVVS Transforms
and Unity Transforms.

Kinemation also supports one of the best features of Entities Graphics, which is
material property overrides. You can still use them, along with
`MaterialMeshInfo` and `RenderMeshUtility`. In fact, aside from LODs and
Unity.Deformations, Kinemation supports the same runtime ECS representation as
Entities Graphics.

Users are often surprised when switching to Kinemation that “everything just
works”.

### Character Ready

Kinemation abstracts away the complexities of skin matrices and deforms your
characters directly from bones. Bones can either be represented as entities, or
as an optimized skeleton just like with Game Objects. A powerful runtime binding
system and nondestructive sockets make it easy to set up character
customization. And characters can leverage additional deformation features such
as blend shapes and dual quaternion skinning.

Kinemation offers animation clip baking, compression, and granular sampling
APIs, backed by the [industry
leading](https://www.youtube.com/watch?v=85uOa2m_kBc) ACL
[library](https://github.com/nfrechette/acl). You’ll find utilities for root
motion, inertial blending, animation masking, and even an EWBIK algorithm (a
multi-target IK solver supporting both position and orientation targets).

With QVVS Transforms, true squash-n-stretch animation is achievable without the
pesky undesirable shearing. Even if you are using Unity Transforms, you can
still take advantage of this feature using optimized skeletons.

The entire pipeline from the binding systems to the compute shaders has been
heavily optimized to handle complex characters. Combining skinned meshes is a
thing of the past. Frustum culling is vertex-accurate, thanks to a special
baking process that analyzes the relationships between vertices, bones, and
blend shapes.

### More Performance for More Things

If you thought the scale and density of environments that Entities Graphics
could handle was impressive, just wait until you see what Kinemation can handle!

Kinemation is faster than Entities Graphics in nearly every aspect, but
especially for dynamic entities and LOD handling. It uses a special culling
system that analyzes the results of frustum culling to minimize the amount of
data that needs to be sent to the GPU each frame. This means the GPU spends less
time waiting for and ingesting data and more time rendering. Both CPU and GPU
performance benefit from this technique. On some projects, this optimization
alone can double framerates.

Submesh sharing is always on in Kinemation, including for skinned meshes. It
uses an intelligent opaque/transparent splitting algorithm to maximize
instancing.

Kinemation comes with a new LOD algorithm that keeps all data cache-friendly
while supporting dynamic scaling of objects at runtime. The memory footprint is
also much smaller than Entities Graphics, helping you pack more entities into
chunks for even more efficiency.

LOD Crossfade helps you push lower resolution LOD meshes closer to the camera
without noticeable popping, which can significantly reduce the number of
triangles the GPU has to draw per frame.

LOD Pack allows packing up to 3 levels (or 2 levels and a fade-out) into a
single entity. This is effectively Mesh LODs, and is one of the fastest LOD
solutions available for spammed entities (especially dynamic ones). LOD Pack has
built-in LOD Crossfade support too. *Note: LOD Pack does not support deforming
meshes.*

Unity 6.2 introduced Mesh LODs, but at the time of writing, Entities Graphics
does not support them. However, Kinemation has fixed that. Mesh LODs are
supported out-of-the-box.

Lastly, if you ever needed to know which entities a frame rendered, Kinemation
provides a feedback flag for this purpose. You can use this for animation
culling or special game logic.

### Truly Extensible for Your Custom Graphics Needs

If you’ve ever tried to create meshes at runtime with Entities Graphics, you
know that it is quite painful. You have to do lifecycle management,
registration, and deal with all the sync points trying to commit changes to it
using the `MeshDataArray` APIs. Wouldn’t it be so much easier if you could just
write to some dynamic buffers, set some config component values, and then not
worry about? Kinemation’s Unique Mesh feature lets you do exactly this. And it
even supports submeshes with submesh sharing.

But what if all you wanted to do was move vertices around for a cloth or
softbody simulation? You might instead look towards Dynamic Mesh, which uses the
deformation pipeline path to support instancing and motion vectors
out-of-the-box.

Got a shader that moves the vertices? You can easily command culling to inflate
the AABBs, including for skinned meshes.

Perhaps you need to procedurally bake your meshes, or setup a different submesh
split setup from what the default bakers do? Kinemation allows you to completely
override renderer baking and comes with a special baking API that allows you to
fully specify the meshes, materials, and render settings for each of your
entities.

But what if you need some really custom graphics? Perhaps involving graphics
buffers and compute shaders? Well you’re in luck, because Kinemation provides
injection points into its powerful round-robin dispatcher allowing you to switch
back and forth between jobs and main thread work without bringing all worker
threads to a halt each time. This comes with access to the culling results so
you can choose to only process visible entities. You can tap into the built-in
graphics buffer broker to get temporary upload buffers as well as growable
persistent buffers. And with some special APIs, you can setup your buffers and
dispatch your compute shaders all from Burst-compiled code.

### Android Performance Notes

On desktop platforms, consoles, and most Apple devices, Kinemation’s rendering
algorithm leverages special GPU hardware caches to perform much of its work.
However, these techniques don’t always pan out correctly on Android devices.
That’s why on Android, many of these optimizations are disabled. But even then,
rendering can still be slower than GameObjects (but never slower than Entities
Graphics). The following discusses the technical reasons for this, and hopefully
gives you some insight into what you can expect or experiment with.

Using Vulkan terminology, Kinemation’s skinned mesh rendering algorithms are
based on Shader Storage Buffer Objects (SSBOs), whereas many Android GPUs prefer
Uniform Buffer Objects (UBOs). Nearly all Mali GPUs that have been on the market
in 2023 and earlier strongly prefer UBOs. Similar for Imagination PowerVR GPUs.

Qualcomm Adreno GPUs like the ones in Meta’s VR headsets are a little different.
Their developer guides suggest using UBOs smaller than 8 kB, and anything larger
should prefer SSBOs. Unity only supports 64 kB UBOs, which means SSBOs are the
best option, and performance tests reflect this. Kinemation’s rendering
outperforms Game Objects on this GPU. However, as vertex count is the main
bottleneck on these GPUs, and skinned meshes tend to require higher vertex
counts for good-looking deformations, Kinemation’s scaling advantage is heavily
offset by the low entity count the hardware can handle. LOD Groups are a
powerful tool for increasing the number of entities possible.

You used to be able to tell what a GPU prefers based on whether it renders a
static ECS scene faster with Vulkan (which uses SSBOs for everything) or OpenGL
ES (which uses UBOs for most things). However, some devices well-suited for
SSBOs may still struggle on Vulkan due to Unity’s Vulkan implementation doing
dumb things sometimes. Adreno GPUs have been susceptible to this.

## Known Issues

-   Kinemation does not support all platforms out-of-the-box. While the ACLUnity
    native plugin is expected to work across all platforms, you may have to
    compile the library yourself for some platforms. If you would like to help
    Kinemation support your target platform officially, please reach out!
    Alternatively, you can add the LATIOS_DISABLE_ACL scripting define at the
    cost of losing animation clip functionality.
-   Occlusion Culling is not supported (not that Entities Graphics supports it
    either).
-   Entities Graphics stats don’t work. Kinemation may provide its own solution
    for this in a future release which will be more extensible and customizable.
    In the meantime, you are strongly encouraged to use RenderDoc to analyze
    what is happening in a frame.
-   In builds when using Unity Transforms, Kinemation does not enforce that a
    skinned mesh’s local transform is set to identity in subsequent frames after
    binding to a skeleton, which may result in incorrect rendering if this value
    is modified later. In the editor, a dedicated system enforces this due to
    incremental baking frequently stomping the data.
-   In a NetCode project, if the skeleton entity is a prespawned ghost, the
    skinned mesh will fail to bind to it as the skeleton will be disabled at
    that time. The current workaround is to make the Game Object with the
    Animator a child of the Game Object with the Ghost Authoring component.
-   If an asset was imported as an optimized hierarchy and internally contains a
    Bone and a SkinnedMesh Renderer named identically, Kinemation may fail to
    bake the skeleton correctly due to a Unity bug/limitation in
    `AnimatorUtility`. Newer Unity versions will warn you about this.
-   Older versions of Unity may delay updates of large Unique Mesh meshes to the
    GPU. This issue has disappeared in Unity 6000.2.8f1 and newer it seems.

## Near-Term Roadmap

-   Sync-less `LatiosEntitiesGraphicsSystem`
-   Automatic Batch Combining
-   Procedural and Indirect Draw Support
-   GPU Deformed Mesh Normals and Tangents Recalculation
-   Blend Shape Animation Baking Helpers
-   Unmanaged Texture APIs
-   Raytracing TLAS with Contribution Culling
