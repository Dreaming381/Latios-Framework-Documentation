# ![](media/60f09d8726e0a13095c19b993f2fb88f.gif)

# Kinemation Animation and Rendering

Kinemation is an animation and rendering solution for Entities which aims to
tightly integrate gameplay and animation for large-scale worlds. Kinemation is
still under active development, and it has only laid down the foundations
towards what it aims to achieve. Yet those foundations are already usable in
projects. As of Unity Entities 1.3, no official Entities-based animation
solution exists.

Check out the [Getting Started](Getting%20Started%20-%20Part%201.md) page!

## Quick Answers

*Can I use Mecanim Animator Controllers?*

Yes. Support is available through an Addon in the Mimic module.

*Do culling and LODs work?*

Yes. They work the way they are supposed to (which is better than Entities
Graphics).

*Is it faster than Game Objects?*

Way faster!

*Will I run out of video memory and cause Unity to crash?*

It is far less likely.

*Does this work with Unity Transforms?*

Yes. Unity Transforms are slower than QVVS, but don’t affect the **big**
optimizations. You’ll still see significant gains and retain the ability to tune
your LODs for stable performance at all zoom levels.

*How did you do this?*

I’ve spent a lot of time on this over the last two years. This is my third
iteration of the design. If you really want to know the details, I encourage you
to reach out to me directly. There’s too much to explain here.

*Does this work in VR?*

Yes. While Android tends to be a mixed bag performance-wise, the most common
headsets have shown preference towards Kinemation’s rendering stack over
GameObjects.

## Features

### Out-of-the-Box Deformations

When you import a Game Object character prefab into your scene, that character
shows up with the pose it was exported with. If you then rotate the Game
Objects, the character deforms.

Well guess what?

When you bake that Game Object to an entity, it remains posed. And when you
rotate the baked entities, the character deforms. It works just like Game
Objects, except way faster!

But there’s more.

With Game Objects you can optimize the hierarchy. Not only can Kinemation bake
these hierarchies, but it will also bake them to its own optimized hierarchy
using the same exported bone settings. But there’s one big difference.
Kinemation provides direct access to the optimized hierarchy buffer, so that you
can always remain in complete control.

Kinemation analyzes your meshes and materials and picks the most likely default
configurations for its wide array of deformation functionality. But you are also
able to customize these configurations if necessary.

And if you think all of that sounds amazing, this next one deserves its own
section!

### Vertex-Accurate Animation-Independent Culling

Classical Unity uses baked animation data of a character to figure out if it is
on screen. But when you start procedurally stretching bones outside the confines
of the baked animations, Unity might think your character is out of view when
your character’s stretched out arms are right in front of the camera.

Kinemation doesn’t care how your character’s bones got to where they are. It
will find a bounding box around all your character’s vertices and not much else.
It does this using a special precomputation step during baking which maps
relationships between vertices and bones. From this, it only needs the bone
positions and a small table to find a suitable bounding box.

You don’t need to do anything. It works automatically using the full potential
of jobs and Burst.

This mechanism also works with blendshapes, even for weights outside the range
of [0f, 1f]. And no. It is not just computing all the vertices. It is doing
something far more efficient.

### A Better Entities Graphics Runtime

A large amount of Entities Graphics has been rewritten to better take advantage
of culling and to support far more characters in open worlds. Now, chunks of
entities that aren’t visible won’t upload data to the GPU. And the skinning
dispatcher will only allocate skinning buffers for the visible entities,
drastically reducing VRAM usage. Compute deform skinning now benefits from LODs.

Both classical Unity and the unaltered Entities Graphics perform skinning
mesh-by-mesh (Entities Graphics at least processes all instances of a mesh at
once). Kinemation handles all meshes for all skeletons at once using a special
compute shader. This removes resource stalls and allows the GPU to use all
available cores to perform skinning as fast as possible and move on to
rendering.

Because of this, Kinemation handles modular characters composed of multiple
meshes and materials incredibly well. Combining meshes is no longer much of an
optimization. Instead, focus on sharing the same base materials and shaders to
improve batching, and check that your skinned mesh chunk occupancy is good. Use
LODs to their full advantage.

A rewritten Entities Graphics might sound really scary, but Kinemation takes
great care to preserve existing workflows and behaviors where they make sense.
It is compatible with all existing shaders written for Entities Graphics, and
material instance properties still work as expected.

But now, the culling process is completely open as public API. You can add your
own culling systems to alter the visibilities of entities without structural
changes or hacks. Or you can react to existing visibility information for
gameplay purposes. You can even mimic what Kinemation does and reserve some
heavy compute shader dispatches for only the entities that will be rendered.
Kinemation even includes APIs to help you manage your own graphics buffers for
custom visual effects.

Kinemation’s culling algorithm is also much more aggressive, using an ECS-based
filtering order that removes many computationally expensive checks early in the
pipeline, all while preserving fancy culling features like shadow map splits,
picking, and highlighting.

One last thing, Kinemation has **no per-frame GC allocations**!

### A Better Entities Graphics Baking Pipeline

It isn’t just the runtime that got a rewrite. Baking also has been rewritten to
take full advantage of the faster runtime and provide more flexibility.

By default, any renderer will be baked using at most two entities. One entity
will store all the opaque materials, while the other will store all the
transparent materials. This means less transforms and less culling work.

However, there may be times you need to override this behavior, such as for
isolating specific materials to specific entities where you want to modify
material property overrides at runtime. Kinemation provides a custom API for
this which allows you to define the sets of meshes and materials for each entity
you bake. That’s right, Kinemation supports entities with multiple meshes!

### LOD Crossfade

One major deviation from Entities Graphics workflow is a completely overhauled
LOD runtime system. This LOD System is designed for the common case where LOD
meshes are spawned as direct children of their LOD Group, and baking will
completely optimize out the LOD Group, removing all component lookups from the
runtime. Having a much smaller archetype footprint and cache-friendly algorithm,
the new LOD implementation gets even better by supporting LOD Crossfade,
including SpeedTree crossfade in Unity 6. This lets you push your lower-res mesh
LODs much closer to the camera without visible popping, allowing you to cram
more entities on screen at once!

### Custom Shader Graph Nodes

Kinemation has more features to offer than what Unity’s built-in Shader Graph
nodes provide. That’s why Kinemation provides its own. Look for the Latios
Vertex Skinning node if you want motion vectors using vertex skinning.

![](media/5342e4cc79a0e5d5b1d9b9b640c467b9.png)

### Dual Quaternion Skinning

You can choose the traditional matrix linear blend skinning or dual quaternion
skinning for each skinned mesh. Dual quaternion skinning preserves volume
better, but can result in sometimes unwanted bulging. Typically, which you use
should match the algorithm used in the DCC application.

### Easy Binding System

Character customization and weapon switching are common use cases that many
other animation packages struggle with. But Kinemation was designed with these
use cases in mind.

No longer do you have to copy magical bone arrays between Skinned Mesh
Renderers. Kinemation bakes transform hierarchy information as strings into Blob
Assets for skeletons and skinned meshes separately. Then at runtime, an
algorithm matches up the skeleton with the mesh to create the binding. This
usually just works without intervention, even for optimized hierarchies. But in
case you have trouble with naming conflicts, you can customize the mapping
strings yourself via optional authoring components.

Otherwise, all you have to do is point the mesh to the skeleton, bone of the
skeleton, or another mesh already bound to the skeleton. The rest happens
automatically in a reactive system.

### Immediate Low-Level (But Still Easy) Animation API

While Kinemation’s rendering stack mimic’s Myri’s philosophy where stuff just
works without code, animation takes the approach of Psyshock where nothing
happens except your code.

During baking, you request which clips to bake and which Animator to bake them
for. You then get a blob asset containing that collection of clips. At runtime
from any thread, with or without the skeleton at hand, you can sample the clip.
You can get the local transform of all bones, or just a single bone. And what
you do with that sampled transform is completely up to you. You can blend it,
discard it, perform some physics analysis on it, or you know, write it to your
entities.

You remain in complete control over animation, and can fully customize it for
your project’s needs. KISS animation is trivial. And if you want to build an
advanced state machine or graph solution, Kinemation provides a great
foundational API and comes with the support of someone all too eager to answer
any questions you may have!

The API is flexible enough for higher-level solutions to be built on top. For
example, the Mimic module has a Mecanim Animator Controller Addon which supports
familiar tools and workflows in native ECS.

### AAA Animation Compression

Unity’s animation compression algorithm is not great. Usually, small sizes come
with great loss of accuracy to the original animation. That was never going to
work for my high-fidelity ambitions. And so, I went about researching possible
alternatives…

And I [found one](https://www.youtube.com/watch?v=85uOa2m_kBc).

Meet [ACL](https://github.com/nfrechette/acl), a solution to animation
compression used by dozens of AAA titles and counting. The entire library is
written with a data-oriented mindset and API and takes full advantage of SIMD
hardware. It was trivial to [wrap a small part of this
library](https://github.com/Dreaming381/AclUnity) and call into it directly from
Burst.

ACL allows for specifying hard limits to quality loss, so that compression
artifacts are imperceptible. Those controls are now within your power whenever
you request an animation clip to be baked into a blob asset. And if you choose
to use the defaults, you’ll get significantly higher quality than Unity’s
defaults, with even smaller compressed sizes.

Now if only Myri had something this good…

### Inertial Motion Blending and Motion Vectors

Kinemation has built-in motion history mechanisms that other features such as
Inertial Motion Blending and Motion Vectors can leverage. Inertial motion
blending is built into Optimized Skeletons and can be controlled with a simple
API. Motion vectors can be used automatically when a supporting shader is used.

### Native Squash and Stretch

Squash and Stretch is one of the fundamental principles of animation. The most
natural way to express this using skeletal animation is to non-uniformly scale
bones. But that doesn’t work in game engines. The issue game engines have is
that the scaling causes rotated child transforms to shear in undesirable ways.

Non-shearing stretch is built into many features of the Latios Framework, and
Kinemation takes full advantage of it to deliver the juiciest of animations.

### Mecanim Controller to Get Started

The Mimic module has a [Mecanim Animator
Controller](../Mimic/Mecanim%20Runtime.md) runtime so that you can continue
using the tools you know. The runtime supports all parameter types, all blend
tree types, transitions, interrupts, root motion, and more. The `MecanimAspect`
provides the API for interacting with the controller at runtime.

Note: The Mecanim module has known issues and is not considered suitable for
production. It is merely provided as a quick-start to get familiar with
Kinemation and also serves as a potential learning resource for how to leverage
Kinemation’s APIs. If you desire a production-ready Mecanim implementation and
are willing to invest in its existence, please reach out!

### Lots of Features

While the above features might be the most impactful, Kinemation has plenty more
to offer. **Blend Shapes** offer an alternative to skeletal animation.
[**Dynamic Meshes**](Dynamic%20Meshes.md) let you animate the vertices directly
in Burst jobs. And the `GraphicsBufferBroker` provides direct access to the GPU
buffers for GPU-based processing. Both exposed and optimized skeletons support
**attachments**, including nested skeletons.

### Performance Guarantee

Except for Android devices which often have crippling GPU limitations,
Kinemation is **fast**! No other ECS animation solution offers the same level
flexibility at the scale Kinemation can achieve. If you aren’t satisfied, report
a bug!

### Android Notes

As for Android, this largely depends on which GPU is in the device. Kinemation’s
skinned mesh rendering algorithms are based on SSBOs, whereas many Android GPUs
prefer UBOs. You can typically tell what a GPU prefers based on whether it
renders a static ECS scene faster with Vulkan (which uses SSBOs for everything)
or OpenGL ES (which uses UBOs for most things). Nearly all Mali GPUs that have
been on the market in 2023 and earlier strongly prefer UBOs. Similar for
Imagination PowerVR GPUs.

Qualcomm Adreno GPUs like the ones in Meta’s VR headsets are a little different.
Their developer guides suggest using UBOs smaller than 8 kB, and anything larger
should prefer SSBOs. Unity only supports 64 kB UBOs, which means SSBOs are the
best option, and performance tests reflect this. Kinemation’s rendering
outperforms Game Objects on this GPU. However, as vertex count is the main
bottleneck on these GPUs, and skinned meshes tend to require higher vertex
counts for good-looking deformations, Kinemation’s scaling advantage is heavily
offset by the low entity count the hardware can handle. LOD Groups are a
powerful tool for increasing the number of entities the hardware can handle.

## Known Issues

-   Kinemation does not support all platforms out-of-the-box. While the ACLUnity
    native plugin is expected to work across all platforms, you may have to
    compile the library yourself for some platforms. If you would like to help
    Kinemation support your target platform officially, please reach out!
-   Occlusion Culling is not supported.
-   Entities Graphics stats don’t work. Kinemation will provide its own solution
    for this in a future release which will be more extensible and customizable.
-   When using Unity Transforms, Kinemation does not enforce that a skinned
    mesh’s local transform is set to identity after binding to a skeleton, which
    may result in incorrect rendering if this value is modified later
-   In a NetCode project, if the skeleton entity is a prespawned ghost, the
    skinned mesh will fail to bind to it as the skeleton will be disabled at
    that time. The current workaround is to make the Game Object with the
    Animator a child of the Game Object with the Ghost Authoring component.

## Near-Term Roadmap

-   EWBIK
-   GPU Deformed Mesh Normals and Tangents Recalculation
-   Blend Shape Animation Baking Helpers
-   Optimized Skeleton Editor Cache
-   Stats and Troubleshooting Diagnostics
-   Cycle-Matching Utilities
-   Blend Tree Helpers
