# Choosing the Right Transform System

*Note: Not all Transform Systems and augmentations detailed in this document are
finished. Each system requires love from people who intend to use it. If the
system you want is unfinished, reach out and get involved!*

The Latios Framework offers several choices for how transforms may work in your
project. Each fundamental system offers several augmentations which may impact
behavior or performance. Transforms affect nearly every aspect of a project.
Picking the correct one is important.

-   Cached QVVS
    -   Extreme Transforms Augmentation
-   Non-cached QVVS (Not yet implemented)
    -   Extreme Transforms Augmentation (Not yet implemented)
-   Unity Transforms (Not yet implemented)
    -   Stretch Compatibility Augmentation (Not yet implemented)
    -   Deterministic Parallel Hierarchy Augmentation (Not yet implemented)
    -   Hierarchy Optimizations? (Requires research)

## Cached QVVS

Cached QVVS is a Transform System built upon the `TransformQvvs` type. It uses a
`WorldTransform` component, a cached `ParentToWorldTransform` component, and a
`LocalTransform` component to always maintain a rough approximation of local and
world transforms for entities within a hierarchy. Root entities only contain the
`WorldTransform` component.

The `DynamicBuffer<Child>` is stored outside ECS chunks. Some entities can be
tagged with `CopyParentWorldTransformTag` to reduce chunk space (especially
useful for meshes with multiple materials).

Excluding `Parent` and `Child` components, a root entity or entity with
`CopyParentWorldTransformTag` requires 48 bytes of transform data, while a child
entity requires 128 bytes.

When working with Cached QVVS, the general philosophy is if you want to read the
`WorldTransform`, just read the component directly. And if you want to write
transforms or read local space, use `TransformAspect` which abstracts hierarchy
details.

### Extreme Transforms Augmentation

This augmentation adds small internal components to child entities in the
hierarchy to keep track of the depth of each entity in the hierarchy. This
allows performing a breadth-first chunk-iterated update for the first 16 levels
of the hierarchy rather than the traditional depth-first recursive update. This
approach has better cache coherency while performing updates, but suffers from a
larger job scheduling overhead and as well as an extra overhead for hierarchy
structure changes (changes in parents). This augmentation is best suited for
large simulations with lots of entity hierarchies, such that the cache savings
outweigh the overhead of the extra jobs.

The extra components add a single byte per child entity plus a 4-byte chunk
component.

## Non-cached QVVS

Compared to its cached counterpart, Non-cached QVVS omits the
`ParentToWorldTransform` and `TransformAspect` in order to reduce memory and
computation cost. However, this may make it more difficult to develop with as
the `WorldTransform` may be more stale and writing transforms requires knowledge
of the hierarchy state by the user.

Entity sizes are identical to cached, except that child entities only require 80
bytes excluding `Parent` and `Child` components.

When working with Non-cached QVVS, the `WorldTransform` is always up-to-date for
root entities and should be both written-to and read-from directly. For child
entities, `WorldTransform` may be stale and should never be written to directly.
Write to `LocalTransform` instead. `WorldTransform` will be updated by the
`TransformSuperSystem`.

### Extreme Transforms Augmentation

This augmentation is identical to the Cached QVVS augmentation of the same name.

## Unity Transforms

Unity Transforms is the solution provided by default ECS. It uses a non-cached
matrix-centric workflow. There is a `LocalTransform` component and a
`LocalToWorld` component present on every moving entity.

The `DynamicBuffer<Child>` has an in-chunk capacity of 16, requiring 144 bytes
of chunk space. `LocalToWorld` is also 16 bytes larger than a QVVS because it
uses a `float4x4` instead of a `float3x4`. The systems may be plagued with
determinism flaws with regards to its hierarchy. Entity in chunk order and
change filters are not deterministic. In addition, because it is matrix-based,
additional computation is required for accessing world-space transform data for
Psyshock, Myri, and rendering with Entities Graphics or Kinemation.

Despite these flaws, Unity Transforms offers the greatest amount of
compatibility with other assets and packages. It also provides better
performance when scaling entities on custom pivots.

Excluding `Parent` and `Child` components, root and child entities each require
96 bytes.

When working with Unity Transforms, `LocalToWorld` may be stale even for root
entities, but is generally the component to read from for world-space
transforms. When writing to transforms, always write to the `LocalTransform`
component.

### Stretch Compatibility Augmentation

This augmentation adds stretch support to Unity Transforms at the cost of 12
extra bytes of memory per entity. Stretch provides an enhanced set of features
for Psyshock and Kinemation.

Despite being a Latios Framework feature, the `Stretch` component is added to
the `Unity.Transforms` namespace for convenience and to avoid conflicts with the
other transform systems. Stretch support replaces the hierarchy update system
with one that is deterministic with regards to change filters.

### Deterministic Parallel Hierarchy Augmentation

This augmentation fixes the determinism issues and performance pitfalls by
replacing the system which handles hierarchy structure changes, instead using
the algorithms that the QVVS systems use. This may exhibit some slight behavior
changes, but most users will likely find these changes beneficial.

### Hierarchy Optimizations?

Overall, optimizing updating the hierarchy is much more difficult, especially
when stretch is involved. This is for two reasons.

First, two lookups are required on a parent instead of 1, which defeats the
Extreme Transforms optimization. This could be remedied at the cost of an
additional 48 bytes per parent entity, which may not be worth it.

Second, stretch is baked into `LocalToWorld`, which means an additional inverse
stretch matrix must be computed and incorporated into the update formula. Such
operation adds significant CPU cost, though that cost relative to the memory
loads remains unknown.
