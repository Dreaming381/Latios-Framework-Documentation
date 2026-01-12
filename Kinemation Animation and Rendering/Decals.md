# Decals

Kinemation has special support for projected decals as pure entities. This
technique has much better performance than the companion Game Object system
Entities Graphics provides.

Note: The shaders provided with this feature are provided as-is without any
long-term guarantee of support. Unity is known for frequently making breaking
changes to their shader stack, and these shaders may break partially or fully
with any editor update.

## Setting Up Projected Decals

First, you will need to open the package manager, select the Latios Framework,
and install the *Kinemation Decal Shaders* sample. These samples add two new
shader graphs along with an hlsl include file. When selecting a shader for a
material, these shaders are found in *Kinemation/Decals/*.

Kinemation decals do not use Unity’s decal projector component for authoring.
Instead, they use normal *Mesh Renderers* with special meshes. The special
meshes can be found in the *Resources* folder inside Kinemation’s package
directory.

![A screenshot of a computer AI-generated content may be
incorrect.](media/b8a5327cc5cd7d460d7a2d0e62bc2fed.png)

These two meshes correspond to orthographic and perspective projection
respectively. Similarly, decal shaders are either orthographic or perspective.
Make sure to use the appropriate mesh with the appropriate shader.

Decals always project along the local forward axis (+Z). Scaling Z will increase
or decrease the projection distance, while scaling X and Y will change the width
and height of the projection respectively. The scale values match world units,
so a Z of 5 means a projection distance of 5 units, while a Y of 2 means a
projection height of 2 units. For perspective projections, the X and Y scale
factors represent the width and height at the maximum projection distance.

![A screenshot of a computer AI-generated content may be
incorrect.](media/88eb58bc716998dc2d82bc488c792fcf.png)

### Layer Masks

Kinemation decals support decal layers. This can be configured through the Mesh
Renderer’s *Rendering Layer Mask*.

### Angle Fade (Not Yet Supported)

Angle fade is an experimental feature of Kinemation decals. Currently, Unity’s
shader infrastructure does not allow this feature to be inserted. However, if
you wish to implement the feature in a custom shader, you can add the *Decal
Settings* authoring component. At runtime, you can add the `DecalAngleFade`
component. Note that URP and HDRP encode different precomputed values and use
different shader code to evaluate them.

## Shader Graph Explanations

While the shader graphs provided are a starting point, you may want to customize
them. This requires you have some understanding of how the shaders work.

Fundamentally, a decal projector shader works by rendering the mesh surface of
the decal, and then for each pixel, determine if an already-rendered pixel from
the depth buffer falls inside the projection using an analytical representation
of the projector mesh. If it does, the projector will compute UV coordinates and
shade over that pixel without overwriting the depth. If it does not, the
projector shader will discard the pixel. (*Note: HDRP shader code mentions that
this might cause artifacts on Apple platforms. If you experience this, you can
modify the shader code to spit out an alpha value instead.*) The custom
functions `ProjectDecalOrthographic` and `ProjectDecalPersepective` are
responsible for doing this analysis and computing the UVs. The UVs have no
scaling and offset applied. A U = 0 is the left side of the projection, while a
V = 1 is the top of the projection. You can apply your own scaling and offset on
top of these values. The `distanceFactor` is a normalized value between 0 and 1
which describes how far away the already-rendered pixel is from the projector. 0
means the pixel is aligned to the projector, while 1 means the pixel is at the
max projection distance.

![A screenshot of a computer AI-generated content may be
incorrect.](media/fe26136f124ef0be02096ffe17380693.png)

Projection only works when the decal projector mesh’s surface is actually
rendered. If the camera were to move inside the mesh, the decal may disappear.
For this reason, Unity’s decal projectors only render back faces and only render
pixels behind the depth buffer values. This however can result in significant
overdraw. Kinemation’s approach is to instead draw front faces with normal depth
testing, and then in the vertex shader, push vertices that lie on the wrong side
of the near clip plane so that they end up right in front of what the camera
sees.

**Warning: Right before release of 0.14.9, I discovered that the shader logic to
perform this has some edge cases I did not account for. This will be resolved in
a future version, but will likely require changes to the custom function
signature.**

Additionally, projection requires the projector normal and tangent baselines are
in the projection direction, which doesn’t match the projector mesh surface.
Another task of the vertex shader is to override the vertex normal and tangent
values such that they fit the projection. All of this vertex shader work is
performed by the `SetupProjectionDecalVertex` custom function.

![A screenshot of a computer AI-generated content may be
incorrect.](media/be0d080d75d44bcedddde989fc778f38.png)

## Deep Hackery

Supporting decal shaders with pure entities unfortunately required some deep
hackery of Unity’s rendering stack to make work. I hope Unity addresses these
issues in the future. But for now, they are documented here.

Kinemation projected decals are actually mesh decals as seen from Unity’s shader
ecosystem. This means that they use mesh decal shader pathways generated by
shader graph, rather than the projected paths. The projected shader pathways are
tied to Unity’s projector component and use classical instancing, hence why they
are not SRP-batcher nor GRD compatible. Meanwhile, mesh decals can be entities
just fine.

### Layer Masks

In URP, the decal rendering layer is read from the classical instancing buffer
or a defaulted-to-zero constant. The custom hlsl file defines an override of
this to source it from the built-in draw call constant. Why Unity can’t do this
when it detects there’s no traditional instancing? I don’t know.

HDRP goes a step further and hides the rendering layer behind the projected path
defines. Hence, the custom projection functions reimplement the masking logic.

### Mesh Modification

For some reason, Unity’s shader graph generators completely ignore the vertex
stage for decals. Kinemation makes an attempt to override this behavior, which
can be found inside FixDecalMeshes.cs inside the Kinemation module. For both URP
and HDRP, this file alters the pass descriptions to enable vertex block
generation.

However, this was not enough. In URP, the decal template still does not generate
critical code needed for vertex modification. So the fix Kinemation employs is
to open that file, and insert the line required if it isn’t already present.
Sometimes, Unity will spit out a shader error about the shaders not compiling.
However, when you select the shaders, they seem to have compiled and rendered
fine. At the time of writing, I do not yet know what causes this.
