# QVVS Transforms

QVVS Transforms is a set of 3D transform mathematics and complementary runtime
system based on a concept of a QVVS representation. A QVVS is a vector-based
representation of an object transform that can account for position, rotation,
and uniform scale in both local space and world space. In addition, it can also
account for a flavor of non-uniform scale in local space referred to as
**stretch**. Stretch influences children positions as one would expect, but does
not influence children in such a way that would introduce shear. This makes QVVS
transforms ideal for representing skeletal squash-and-stretch animations or
physics-based elastic compounds.

Check out the [Getting Started](Getting%20Started.md) page!

## Features

### QVVS Types and Operations

The module defines the basic struct types `TransformQvvs` and `TransformQvs`
which you can use in any of your own types regardless of which flavor of
transform system you use. For math operations using these transform types, look
no further than the static `qvvs` class.

### Always Up-To-Date Transforms

The QVVS Transform System is designed to keep the full hierarchy in-sync at all
times, similar to how `UnityEngine.Transform` works. This is different from
other ECS transform systems which require a dedicated system to explicitly
synchronize the hierarchy.

Always up-to-date transforms means more freedom in the order systems can execute
without worrying about stale data. And it eliminates overhead of inactive
hierarchies, making it especially effective in rollback environments.

### Hierarchy-First Design

Because of the always up-to-date mechanisms, hierarchies are immediately
traversable the moment an entity is instantiated. Additionally, the APIs make it
easy to obtain exclusive write access to all entities in a hierarchy within a
parallel job, making many gameplay systems easier to build and parallelize.

### Motion History

Motion History allows for tracking world transforms of previous frames. This can
be used for various things, such as position-based dynamics, kinematic
platforms, or motion vectors.

### Inheritance Flags

Inheritance Flags allow for specifying the various properties of a child
transform that should be “locked” to world-space values during propagation. This
brings a greater level of expressiveness to ECS transforms.

### GameObjectEntity

The `GameObjectEntity` mechanism allows for specifying non-subscene
`GameObject`s as entities with automatic transform syncing. They can even be
assigned a subscene “host” entity to combine baked entity data and runtime
`GameObject` data all into a single entity. It is especially useful for things
like the main camera.

### Chunk Efficiency

QVVS Transform systems try to be as efficient as possible when it comes to chunk
storage. Dynamic buffers are stored outside the chunk. The `WorldTransform`
component is only 48 bytes, and is sufficient for solo entities for physics and
rendering. Root entities with children will typically use 64 bytes (80 in the
worst-case). And child entities only require 60 bytes total. This all contrasts
to Unity Transforms which use 64 bytes for `LocalToWorld` plus 32 bytes for
`LocalTransform`.

### Deterministic and High Performance

Unity’s Transform systems suffer from little bugs which break determinism of
change filters, chunk ordering, and child buffer ordering when dealing with
complex dynamically-changing hierarchies. Not only do QVVS transform systems fix
these issues, but they do so using special optimization techniques to outperform
Unity’s systems as well.

### Baking Ready

QVVS Transforms supplies baking systems that work out-of-the-box and respect
`TransformUsageFlags`. In addition, special features can be controlled via
[special baking components workflow](QVVS%20Transforms%20Baking.md) that even
works when multiple bakers try to add the same feature.

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

### Unity Transforms Compatibility Mode via Abstract Aspects

The QVVS Transforms module provides an Abstract namespace which the other
modules use for compatibility with both QVVS and Unity Transforms. While not all
features are supported with Unity Transforms, the mode offers a solution for
those who need better compatibility with other offerings.

## Known Issues

-   The Entities Hierarchy does not show hierarchy relationships for closed
    subscenes.
-   Some representations of Unity Transforms don’t map to QVVS cleanly in
    abstract APIs, and information may be discarded in both directions.
-   The always up-to-date transform system requires a bit more boilerplate to
    use than might be expected. This is primarily due to limitations in Unity’s
    ECS APIs.
-   Instantiating live entities requires some extra care, as otherwise an
    instantiated entity might retain its reference to a hierarchy slot it
    doesn’t belong to.
-   Modifying authoring transforms while in play mode does not work correctly
    currently, as Unity’s live patcher can desynchronize the transform
    hierarchy. Editor World systems which modify transforms may also cause
    problems for the same reason.

## Near-Term Roadmap

-   Fixed-Rate Transforms
-   Improved live baking support
-   More transform writing APIs
-   Improved support for instantiating child entities
