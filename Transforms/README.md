# QVVS Transforms

QVVS Transforms is a collections of 3D transform systems based on a concept of a
QVVS representation. A QVVS is a vector-based representation of an object
transform that can account for position, rotation, and uniform scale in both
local space and world space. In addition, it can also account for a flavor of
non-uniform scale in local space referred to as **stretch**. Stretch influences
children positions as one would expect, but does not influence children in such
a way that would introduce shear. This makes QVVS transforms ideal for
representing skeletal squash-and-stretch animations or physics-based elastic
compounds.

Check out the [Getting Started](Getting%20Started.md) page!

## Features

### QVVS Types and Operations

The module defines the basic struct types `TransformQvvs` and `TransformQvs`
which you can use in any of your own types regardless of which flavor of
transform system you use. For math operations using these transform types, look
no further than the static `qvvs` class.

### Chunk Efficiency

All flavors of transform systems try to be as efficient as possible when it
comes to chunk storage. Dynamic buffers are stored outside the chunk. Entities
that purely copy their parent world transform replace their local transform
component with a tag component. And entities without parents or children can
function with a single world transform component as small as 48 bytes that is
sufficient for physics and rendering.

### Deterministic and High Performance

Unity’s Transform Systems suffer from little bugs which break determinism of
change filters, chunk ordering, and child buffer ordering when dealing with
complex dynamically-changing hierarchies. Not only does this module fix these
issues, but it does so using special optimization techniques to outperform
Unity’s systems as well.

But if the baseline performance isn’t enough, there are multiple algorithms
provided which are each suited for different simulation scales, especially at
very high entity counts.

### Archetype Correction

No matter what weird entity archetypes you throw at the system, the transforms
will correct themselves to be a valid configuration, and do the most intuitive
thing. It doesn’t matter if you set a `Parent` to `Entity.Null` or remove it
altogether. Both will result in the Entity being parentless after the transform
system updates, and the entity won’t even teleport. If you add a `Parent`
component but no `LocalTransform`, the `LocalTransform` will be set to identity.
But if the `LocalTransform` is already there, it will be left untouched.

### Motion History

Motion History allows for tracking world transforms of previous frames. This can
be used for various things, such as position-based dynamics, kinematic
platforms, or motion vectors.

### Baking Ready

All transform system flavors supply baking systems that work out-of-the-box and
respect `TransformUsageFlags`. In addition, special features can be controlled
via [special baking components workflow](QVVS%20Transforms%20Baking.md) that
even works when multiple bakers try to add the same feature.

### Optimizations Everywhere

It isn’t just the transform systems themselves that are faster. Everything
working with transforms can benefit greatly from the improved representation.
Psyshock and Myri can extract world-transform information easily without having
to dissect matrices. And Kinemation benefits from significantly cheaper triangle
winding calculations and uploads the QVVS transforms directly to the GPU where
they are transformed during the sparse uploader compute shader. As this shader
is memory-bound, the conversion to matrix is effectively “free”. The CPU never
has to compute a matrix. In addition, the upload makes better use of SIMD
hardware.

## Known Issues

-   Compatibility with many other packages outside of the Latios Framework is
    currently broken. Support is planned, but requires discussion and
    participation from the community.
-   The Entities Hierarchy does not show hierarchy relationships for closed
    subscenes.

## Near-Term Roadmap

-   Uncached Transforms
-   Unity Compatibility Transforms
-   IJobEntityHierarchy
