# Psyshock’s Collision Engine Guide for Contributors

This document outlines the architecture of Psyshock’s collision queries. Refer
to this guide if you wish to:

-   Add a new collider type
-   Add a new query method
-   Fix existing bugs
-   Improve the efficacy and performance of a query

This guide does **not** discuss the mathematics in depth. Nor will it cover the
broadphase algorithms, as those are well-covered in the [Optimization
Adventures](../../Optimization%20Adventures/Part%201%20-%20Find%20Pairs%201.md)
series. Psyshock relies primarily on four commonly-known concepts and algorithms
for queries:

-   Separating Axis Theorem (SAT)
-   Minkowski Differences
-   Voronoi Regions of Shapes (not to be confused with Voronoi diagrams)
-   Gilbert-Johnson-Keerthi (GJK)
-   Minkowski Portal Refinement (MPR)

There are plenty of materials on the internet for the above topics. Please
familiarize yourself with them.

In addition, this guide will also assume you are familiar with collision
manifolds, which are discussed in the [Getting Started
Guide](../Getting%20Started%20-%20Part%206.md). You might also wish to read
[this](../../Tech%20Adventures/Part%204%20-%20Road%20to%20Physics%20Simulation%202.md)
to familiarize yourself with some of the mathematics, though as Psyshock
continues to improve, some of the details may be out-of-date.

## Colliders

Psyshock supports several types of colliders. Currently, each type falls into
one of two categories:

-   Convex
-   Composite

The key difference between the two is that a composite collider is actually a
collection of embedded convex colliders, typically accompanied by some sort of
acceleration structure. These embedded convex colliders are referred to as a
*subcollider*, and each have a designated index within the composite collider. A
convex collider does not have subcolliders, and will always use a subcollider
value of 0 for queries.

Collider types use an enumeration that orders them as follows:

-   Convex
    -   Sphere
    -   Capsule
    -   Box
    -   Triangle
    -   Convex (Mesh)
-   Composite
    -   TriMesh
    -   Compound
    -   Terrain

TriMesh and Terrain colliders exclusively store triangles. In the case of a
TriMesh, the collider is *hollow*. That is, a small sphere fully inside a
water-tight TriMesh will be considered **not intersecting**. Psyshock doesn’t
know whether or not a TriMesh is water-tight. Perhaps when it is able to
determine this, this behavior can change.

Similarly, a small sphere fully underneath a terrain will also be considered not
intersecting.

### Collider Union

The `Collider` type is able to represent every unique collider type. This works
via the concept of a union, where the internal memory space of a `Collider` is
aliased for each type.

The first 4 bytes of the collider store the 1-byte type `enum`, specifying which
type of collider the `Collider` stores. The reason a full 4 bytes is reserved is
because the `enum` is the first value needed in every algorithm, and is the
first to be available during a cache load. Additionally, the next values at byte
offset 4 require 4-byte alignments.

The last 8 bytes are reserved for a `BlobAssetReference`, specifically for types
that require it. Byte offsets 4 – 39 are for whatever a specific type needs.

This means that types which require a blob asset are stored in `Collider` with
an offset 48 minus the specific type size, with a max size of 44 bytes.
Meanwhile, types that do not use a blob asset are stored in `Collider` starting
at offset 4 and can be up to 36 bytes in size.

Types that do not use a blob asset have an internal named field inside
`Collider`. However, only `ConvexCollider` has a named field, as having multiple
named fields containing a `BlobAssetReference` at the same offset causes Unity’s
blob asset serializer to crash (or at least, it used to). For TriMesh, Compound,
and Terrain collider types, the `m_convex` field is reinterpreted via internal
extension methods.

Consequently, types using blob assets all currently have a starting offset of 16
bytes and are 32 bytes in size. But it is possible to change this in the future.
This is the resulting layout:

|    | Sphere      | Capsule     | Box        | Triangle | Convex  | TriMesh | Compound    | Terrain          |
|----|-------------|-------------|------------|----------|---------|---------|-------------|------------------|
| 0  | enum        | enum        | enum       | enum     | enum    | enum    | enum        | enum             |
| 4  | center.x    | pointA.x    | center.x   | pointA.x |         |         |             |                  |
| 8  | center.y    | pointA.y    | center.y   | pointA.y |         |         |             |                  |
| 12 | center.z    | pointA.z    | center.z   | pointA.z |         |         |             |                  |
| 16 | radius      | radius      | halfSize.x | pointB.x | scale.x | scale.x | scale       | scale.x          |
| 20 | stretchMode | pointB.x    | halfSize.y | pointB.y | scale.y | scale.y | stretch.x   | scale.y          |
| 24 |             | pointB.y    | halfSize.z | pointB.z | scale.z | scale.z | stretch.y   | scale.z          |
| 28 |             | pointB.z    |            | pointC.x |         |         | stretch.z   | baseHeightOffset |
| 32 |             | stretchMode |            | pointC.y |         |         | stretchMode |                  |
| 36 |             |             |            | pointC.z |         |         |             |                  |
| 40 |             |             |            |          | Blob    | Blob    | Blob        | Blob             |
| 44 |             |             |            |          | Asset   | Asset   | Asset       | Asset            |

## Collider Simple Operations

When adding a new collider type, there are quite a few places you need to touch.
Let’s start with the simple ones. Remember that for in-progress collider types,
it is fine to leave internal implementations throughout the codebase. But you
should not add the collider type to a `switch case` or make it castable to/from
`Collider` until the new type is ready.

### Casting

Each public collider type has a defined pair of `implicit` casts to and from
`Collider` inside ColliderPsyshock.cs. Casting to a `Collider` should always be
safe. Casting from a `Collider` requires a safety check to ensure the type
matches. The safety check has a `switch case` you need to add to.

### Scale and Stretch

The `Physics` class defines a method `ScaleStretchCollider()` inside
Physics.ScaleCollider.cs, which comes in two `public` overloads. The first
overload passes a `Collider` by `ref`, and modifies it in place. The second
passes a `Collider` by value and returns a modified copy (by calling the first
overload).

However, there’s actually an `internal` pair of overloads following the same
pattern for each specific collider type. If you add a new collider type, you
will need to make a new pair of overloads. When the collider type is ready to
become public API, you will also need to add a `switch case` handler inside the
`public` `ref` overload that invokes your internal `ref` implementation.

### Aabbs

The `Physics` class defines a method `AabbFrom()` inside Physics.Aabbs.cs, in
which similar to the above, there are a pair of methods you must implement for
your new type. The first method takes your collider type and a `RigidTransform`
by `in` parameters. The second method takes your collider type without `in`, and
a `TransformQvvs` by `in`. In this second method, you should call
`ScaleStretchCollider()`, and then call the `RigidTransform` overload. This
pattern of applying a `TransformQvvs`’s scale and stretch into the collider, and
then working with `RigidTransform`s is a common paradigm in Psyshock. There are
two `switch case` methods you must update with your new type in this file.

## Point and Ray Queries

Inside Psyshock’s *Internal/Queries/PointRayCollider* folder, you will find a
PointRay\<ColliderType\>.cs file for each collider type. Each of these files
defines an `internal static class` of the same name.

Each of these static classes must implement two specific methods.

```csharp
public static bool DistanceBetween(float3 queryPoint, in <ColliderType>Collider collider, in RigidTransform colliderTransform, float maxDistance, out PointDistanceResult result) { }
public static bool Raycast(in Ray ray, in <ColliderType>Collider collider, in RigidTransform colliderTransform, out RaycastResult result) { }
```

The class may define additional methods, and can optionally make them `public`
if they are to be used by other Psyshock internal methods outside the class.

### DistanceBetween()

The `DistanceBetween()` algorithm evaluates the nearest point on the collider’s
surface relative to the `queryPoint`, and then returns info about that nearest
point.

If the collider type has a concept of “inside” (some collider types like
triangles don’t), then if the `queryPoint` is inside, the
`PointDistanceResult.distance` represents the minimum distance the `queryPoint`
must move to be on the surface of the collider. Then the
`PointDistanceResult.distance` is negated so that the result is negative. The
`PointDistanceResult.hitpoint` is the point on the surface of the collider the
`queryPoint` would have to move to.

If the `queryPoint` is outside or on the surface of the collider, then the
`PointDistanceResult.hitpoint` is the closest point on the surface of the
collider to the `queryPoint`, and the `PointDistanceResult.distance` is the
distance between the `queryPoint` and the `PointDistanceResult.hitpoint`.

`PointDistanceResult.normal` is a normal vector pointing outward from the
collider at `PointDistanceResult.hitpoint`. If the hitpoint is on a hard edge,
this normal should be the average normal of the two adjacent faces. If the
hitpoint is a vertex, this normal should be the average normal of all the edges
which share the vertex.

Be very careful about classifying the hitpoint as being on a vertex or hard edge
correctly. Incorrectly classifying or failing to classify such cases is a common
cause of simulation instability. 32-bit floating point numbers can make this
especially challenging.

The `DistanceBetween()` method should return `false` when
`PointDistanceResult.distance` is greater than `maxDistance`. In such cases, the
values of `PointDistanceResult` can be assigned anything.

### Raycast()

The `Raycast()` algorithm tries to find an intersection of a ray starting
outside the collider, and penetrating into the colliders surface. `Raycast()`
should always return `false` if the ray starts inside or on the surface of the
collider. It should also return false if the ray travels the `maxDistance`
before reaching the collider’s surface.

`RaycastResult.distance` is the distance the ray traveled before hitting the
collider surface. `RaycastResult.hitpoint` is the point on the collider where
the ray hit. `RaycastResult.normal` is the normal at the `hitpoint`, with the
same rules and cautions as `PointDistanceResult.normal`.

`RaycastResult` can be assigned anything if the method returns `false`.

For both of these methods, the subcollider should be 0 for non-composite types,
and the subcollider index of the convex collider with the minimum distance
(higher magnitude negative values are preferred over everything else) for
composite types, where performing the query on the subcollider directly would
produce a `true` value.

### PointRayDispatch

The `PointRayDispatch` static class located in PointRayDispatch.cs is
responsible for forwarding the public API query methods to the collider
type-specific classes. There are two `switch case` methods that match the
signatures of the specialized methods, except that they accept a `TransformQvvs`
instead of a `RigidTransform`. Therefore, these methods are responsible for
calling `ScaleStretchCollider()` as well.

Additionally, there is a third `switch case` method that accepts a specific
subcollider index. For composite types, this method implements the logic to
target the specific subcollider directly in the `case` block. The
`RaycastResult.subColliderIndex` is patched by the caller of this method, so it
does not need to be set in this method.

## Collider Collider Queries

Most of the time, queries take place between colliders. Psyshock specializes
every combination of collider types. These specializations may choose to
leverage general-purpose algorithms if appropriate, but always have the option
to implement something custom for better performance or greater robustness.

The specializations are located in the *ColliderCollider* folder next to the
*PointRayCollider* folder. The specializations are for combinations, not
permutations. The lower `enum` value is always listed first. Therefore, there is
a `SphereSphere`, and a `SphereCapsule`, but no `CapsuleSphere`. The file names
match the `internal static class` names.

The following are the methods each class must implement. Similar to above, each
class can also implement other public and private methods to be used internally.

```csharp
public static bool DistanceBetween(in <HigherEnumType> higherCollider,
                                   in RigidTransform higherTypeTransform,
                                   in <LowerEnumType> lowerCollider,
                                   in RigidTransform lowerTypeTransform,
                                   float maxDistance,
                                   out ColliderDistanceResult result)
{ }

// Only required if the type with the higher type enum value is a composite collider
public static unsafe void DistanceBetweenAll<T>(in <HigherEnumCompositeType> higherEnumCollider,
                                                in RigidTransform higherEnumTransform,
                                                in <LowerEnumType> lowerEnumCollider,
                                                in RigidTransform lowerEnumTransform,
                                                float maxDistance,
                                                ref T processor) where T : unmanaged, IDistanceBetweenAllProcessor
{ }

public static bool ColliderCast(in <LowerEnumType> colliderToCast,
                                in RigidTransform castStart,
                                float3 castEnd,
                                in <HigherEnumType> targetCollider,
                                in RigidTransform targetTransform,
                                out ColliderCastResult result)
{ }

// This one is only needed if LowerEnumType != HigherEnumType
public static bool ColliderCast(in <HigherEnumType> colliderToCast,
                                in RigidTransform castStart,
                                float3 castEnd,
                                in <LowerEnumType> targetCollider,
                                in RigidTransform targetTransform,
                                out ColliderCastResult result)
{ }

public static UnitySim.ContactsBetweenResult UnityContactsBetween(in <HigherEnumType> higherCollider,
                                                                  in RigidTransform higherTypeTransform,
                                                                  in <LowerEnumType> lowerCollider,
                                                                  in RigidTransform lowerTypeTransform,
                                                                  in ColliderDistanceResult distanceResult)
{ }
```

Details about the requirements of these methods will be discussed later in this
article.

### ColliderColliderDispatch

Similar to `PointRayDispatch`, the `ColliderColliderDispatch` static class is
responsible for forwarding public API methods to the specialized static classes.
Additionally, this class resolves permutations, flipping the roles of colliders,
and unflipping results. And it deals with differences between composite and
non-composite colliders.

Because this class dispatches pair combinations, it has many combinations it
needs to work with, and that code can be very tedious to write by hand.
Therefore, `ColliderColliderDispatch` uses T4 to generate the file. The source
file to modify is ColliderColliderDispatch.tt, and the generated file is
ColliderColliderDispatch.gen.cs.

When adding a new collider type, you only need to modify the `Dictionary`
instances at the top of the file, and maybe adjust the `firstComposite` index.
Then regenerate the C\# file. Adding a new query method can be more involved.
You will need to study how the existing methods are generated and try to
replicate them.

## Collider Collider DistanceBetween()

The `DistanceBetween()` method is the most critical method, and also the most
bug-prone. There are certainly bugs in the existing implementations where the
following rules are violated.

The method is similar to `DistanceBetween()` for a point, except it is finding
points on both colliders. If the colliders are intersecting, the
`ColliderDistanceResult.distance` is a negative value representing how far one
of the colliders would need to move apart in order to only be touching.
`hitpointA` and `hitpointB` represent where on the two collider surfaces the
colliders would just be touching, but without actually moving the colliders.

If the colliders are not intersecting, then `ColliderDistanceResult.distance` is
a positive value and `hitpointA` and `hitpointB` represent the two closest
points.

In either case, it is possible that there are multiple valid pairs of
`hitpointA` and `hitpointB`. In such cases, there are specific rules that must
be followed to ensure simulation stability.

If the two shapes involve hard edges and vertices, it may be possible that two
faces – one from each collider – are parallel and represent an infinite number
of closest point pairs. In this case, `DistanceBetween()` should try to identify
a vertex on one face that projects onto the other face, and use that for the
result point pair. If no vertex projects, then that means two edges “cross”, and
there is a closest point pair between two edges.

Similarly, it may be possible that an edge is closest to a face. In this case,
if either endpoint of the edge projects onto the face, use that endpoint vertex
for generating the result pair. Otherwise, an edge along the face must “cross”
the near edge, and there is a closest point pair between those two edges.

### Feature Codes

`ColliderDistanceResult` has two internal 16-bit values `featureCodeA` and
`featureCodeB`. These codes represent the “features” that the closest points lie
on. The two most-significant bits represent the type of feature:

-   0 = vertex
-   1 = edge
-   2 = face
-   3 = invalid

For rounded shapes, this feature code maps to the nearest interior geometry that
the surface is curved around.

The remaining 14 bits represent a unique ID for a given feature type. For
composite colliders, this is relative to the specific embedded convex
subcollider.

### General Helpers

A common practice to help obtain accuracy is to reframe the pair of colliders in
relative space. That is, pretend that one collider has an identity transform,
and reposition the other accordingly (the B collider in A’s space). Psyshock’s
algorithms do this, and store the result into a `ColliderDistanceResultInternal`
type. Afterwards, they call `InternalQueryTypeUtilities.BinAResultToWorld()` to
go back to a world-space result.

Psyshock includes a port of Unity Physics GJK and EPA algorithms, included in
the `GjkEpa` class. This algorithm works with all convex collider types.
However, it is a finite support solver. That is, the number of possible support
shapes must be finite for an infinite number of directions. For rounded shapes
such as spheres and capsules, it operates as if they had a radius of 0, and then
subtracts the radius from the final distance.

The result of GJK and EPA will produce indices for up to 3 vertices per
collider. Each index is a byte, so colliders may only have up to 256 vertices
each. The algorithm may produce face-face pairs and face-edge pairs. It may even
produce an edge that cuts through a face. This is a common cause of simulation
bugs, and at the time of writing, not all code paths using GJK+EPA handle these
results elegantly. EPA also has lower accuracy, and can add up to 1e-3 of
positional error into the result (although it is usually closer to 1e-4). This
is most noticeable when debugging with “perfect inputs”. The error comes from
the integer quantization it uses, as the algorithm operates using integers
rather than floating-point values.

### Algorithms in Use

The following table breaks down the algorithms used by pairs of convex
colliders.

|          | Sphere                   | Capsule                                              | Box                                      | Triangle                                 | Convex                                   |
|----------|--------------------------|------------------------------------------------------|------------------------------------------|------------------------------------------|------------------------------------------|
| Sphere   | distance between centers | distance between center and nearest point on segment | Voronoi region sphere center lies within | Voronoi region sphere center lies within | Voronoi region sphere center lies within |
| Capsule  |                          | distance between nearest segment points              | SAT                                      | custom decision tree                     | GJK+EPA                                  |
| Box      |                          |                                                      | SAT                                      | GJK+EPA                                  | GJK+EPA                                  |
| Triangle |                          |                                                      |                                          | GJK+EPA                                  | GJK+EPA                                  |
| Convex   |                          |                                                      |                                          |                                          | GJK+EPA                                  |

### DistanceBetweenAll()

Composite colliders require a `DistanceBetweenAll()` implementation. Instead of
returning a `true`/`false` value and outputting a `ColliderDistanceResult`, they
must call `Execute()` on the passed-in `processor` whenever a subcollider would
produce a `true` value for `DistanceBetween()`. The most critical part is that
the `processor` should never ever be copied. It must always be passed around by
`ref`, or some algorithms may capture a pointer to it to embed in other data
types.

Most composite colliders will have some form of mid-phase algorithm to cull out
colliders based on bounding-box checks.

## ColliderCast

`ColliderCast()` is ironically easier to understand, but more sensitive to
performance optimizations. The critical part is to identify how far a collider
must move in a given direction before it touches the other collider. After that,
the rest of the results can be populated by moving the collider that distance,
and then calling `DistanceBetween()`. Convention is to use the target collider’s
hitpoint as the `ColliderCastResult.hitpoint`.

A valid trick is to move the target collider opposite to the caster instead of
moving the caster. The distance produced will be the same, and then the caster
can be moved for the `DistanceBetween()` step. Psyshock currently uses this
trick, but the code could probably be refactored better.

While `DistanceBetween()` has the general-purpose GJK+EPA algorithm,
`ColliderCast()` implementations can rely on MPR. However, Psyshock’s
implementation of MPR operates on finite support points. And unlike GJK+EPA, it
can’t easily factor in radial offsets into the results. So MPR only supports
box, triangle, and convex mesh colliders.

Below is the table of algorithms Psyshock uses for `ColliderCast()` currently.

| Caster\\Target | Sphere                                                        | Capsule                                                                                                        | Box                                                                                     | Triangle                                                                                     | Convex                                                                                         |
|----------------|---------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Sphere         | raycast inflated sphere                                       | raycast inflated capsule                                                                                       | raycast rounded box                                                                     | raycast rounded triangular prism                                                             | raycast inflated convex, then spherecast edges                                                 |
| Capsule        | reverse raycast inflated capsule                              | raycast rounded parallelogram prism                                                                            | raycast capsule endpoints against inflated box, then capsule-cast against all box edges | sphere-cast capsule endpoints against triangle, then capsule-cast against all triangle edges | sphere-cast capsule endpoints against convex, then capsule-cast against all edges with culling |
| Box            | reverse raycast rounded box                                   | reverse raycast capsule endpoints against inflated box, then reverse capsule-cast against all box edges        | MPR                                                                                     | MPR                                                                                          | MPR                                                                                            |
| Triangle       | reverse raycast rounded triangular prism                      | reverse sphere-cast capsule endpoints against triangle, then reverse capsule-cast against all triangle edges   | MPR                                                                                     | MPR                                                                                          | MPR                                                                                            |
| Convex         | reverse raycast inflated convex, the reverse spherecast edges | reverse sphere-cast capsule endpoints against convex, then reverse capsule-cast against all edges with culling | MPR                                                                                     | MPR                                                                                          | MPR                                                                                            |

## Unity Contacts

`UnityContactsBetween()` is a key algorithm in rigidbody simulation. The
algorithm takes in the pair of colliders, as well as an already-produced
`ColliderDistanceResult` between them. Contacts are typically generated where
vertices touch the opposite collider’s faces, and where edges cross.

A crucial part of this algorithm, and often the biggest failure point is
selecting a reasonable contact normal. Bad contact normals can cause the
algorithm to think the contacts are deeply penetrating, causing explosive
reactions. Additionally, a bad contact normal can prevent the algorithm from
finding contacts, resulting in missed collisions. Though usually there’s a
fallback to using the closest points as a contact.
`ContactManifoldHelpers.GetSingleContactManifold()` can produce this singular
contact.

## Blob Assets

Several collider types use blob assets. Their internal structures need to be
understood when working with the internals of Psyshock.

### Convex Colliders

The convex mesh colliders store a blob asset containing the unscaled convex hull
mesh. The convex hull has strict limitations of using only up to 252 vertices
and faces, though the edge count can be higher. These limitations come from
Unity Physics, and ensure that the EPA algorithm has a maximum number of
iterations.

The data structure of the blob asset provides faces, indices into edges within
each face, and indices of vertices within each edge. Additionally, vertices and
edges have backlinks back to the faces, but vertices do not backlink to the
edges. There’s also dedicated 2D data structures for when stretch has a single
axis set to 0. However, the codepaths using these data structures are not
well-tested.

In the future, it may be worth switching the data structure to a half-edge
structure, which is more compact. However, the half-edge structure is not
immediately intuitive how to navigate it, as its properties are not
self-advertising.

### TriMesh Colliders

TriMesh colliders store an array of triangle colliders, sorted by the mid-phase
structure. Each element has an index into the source array of triangles the
collider was built from.

Currently, the mid-phase structure is a bunch of AABBs sorted by minimum x-axis
values, with a complementary interval tree along the x-axis. This is the same
data structure as a `CollisionLayer` using only a single bucket. This data
structure is cheap to build, and has generally proven to be sufficiently fast so
far. But there are definitely more efficient structures available. BVHs are a
popular choice. But a 27-way axis partitioning tree could potentially be a
better fit for the distribution of triangles in a mesh.

### Compound Colliders

Compound colliders currently can store sphere, capsule, and box collider
primitives inside. The subcolliders should be individually retrieved by calling
`CompoundCollider.GetScaledAndStretchedSubCollider()`. Each subcollider has a
local rigid transform. The position of this transform is referred to as an
*anchor*. The blob stores a bounding box around all the anchors, as well as the
maximum extent any subcollider extends away from its anchor. The mid-phase
structure is thus a bunch of bounding spheres all of the same radius, and that
radius can be scaled on the fly to adjust for scale and stretch. The bounding
spheres are sorted by their centers along the x-axis, allowing for queries to
binary search for start and end ranges, and sweep over a subset of the
subcolliders. In practice, the very cheap y-axis and z-axis tests against the
bounding sphere are what make this mid-phase effective, especially when the
subcolliders are spread out and relatively the same size (such as voxels). But
there’s definitely opportunity for a better mid-phase.

### Terrain

Terrain colliders are probably the most thought-out and optimized composite
collider types. The reason for this is that many spatial queries may hover over
a large swath of the terrain, overlapping the full terrain’s AABB, but never
actually intersecting any of the terrain triangles.

The terrain blob stores the 16-bit heightmap. But in addition, it has a
hierarchy of patches, where a patch is an 8x8 grid. At the leaf-level, each cell
references two terrain triangles, and stores the orientation of the diagonal
that splits the cell into two triangles, as well as a pair of bits to mask out
triangles treated as holes. At all levels of the hierarchy, each cell also
includes min and max height values. Some special SIMD code can test an AABB
against all cells in a patch in one shot. However, the iteration is modularized
such that in the future, algorithms can further refine the set of cells using
shapes other than AABBs.

## Future Steps Needed for Psyshock Collision

Right now, the biggest issue for Psyshock is that sometimes the math glitches
out, and contacts don’t get evaluated correctly. The code has become somewhat
obfuscated due to the extreme number of edge-cases that came up in testing. Some
of the code came from a time when I tried to replicate Unity Physics behavior as
closely as possible, which wasn’t good either. And some of it came from a time
before newer tools in the framework were ready. A lot of code is repetitive in a
lot of places, where it would be better to be refactored into internal
general-purpose utilities and let Burst work its magic. It has been a learning
experience.

I think refactoring common functionality into internal utilities will help
isolate the bugs present. Additionally, I think we need a tool that can spawn
and simulate a physics scene, and then record each simulation step and replay it
for debugging. This way, we can much more quickly catch issues and fix them.

Eventually, I would like to phase out the Unity Physics algorithms (or at least
make them optional) in favor of algorithms that are better documented and
understood by the community.

Psyshock has a lot of potential, but it needs the help of everyone to realize
it. If you’d like to help, please reach out on the framework’s discord!
