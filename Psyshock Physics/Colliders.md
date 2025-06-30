# Colliders

Psyshock supports the following collider types:

-   Sphere Collider
    -   center and radius
-   Capsule Collider
    -   pointA, pointB, and radius
-   Box Collider
    -   center and halfwidth
-   Triangle Collider
    -   pointA, pointB, and pointC
    -   no built-in authoring component
-   Convex Collider
    -   local-space blob and non-uniform scale factor
-   TriMesh Collider
    -   local-space blob and non-uniform scale factor
-   Compound Collider
    -   local-space blob and uniform scale factor
-   Terrain Collider
    -   local-space heightmap blob of signed 16-bit integers and a hole mask
    -   a base height offset integer, and a non-uniform scale factor
    -   triangles are always in the positive xz quadrant in local space
    -   experimental, with no baking of the `UnityEngine.TerrainCollider` at
        this time

## Authoring

Custom authoring components for all Psyshock colliders are still a
work-in-progress. Use the following components for each collider type and ensure
the component is enabled:

### Sphere Collider

Use the PhysX *Sphere Collider*.

![](media/edee1c765d995b12f65bff2dfb3a5d35.png)

### Capsule Collider

Use the PhysX *Capsule Collider*.

![](media/85df620e9bdc341d5b9d0978112189c1.png)

### Box Collider

Use the PhysX *Box Collider*.

![](media/87073ef01be44817f3ceed69d82cfdeb.png)

### Convex or TriMesh Collider

Use the PhysX *Mesh Collider*. If you want a Convex Collider ensure *Convex* is
checked. Otherwise, a TriMesh Collider will be baked instead.

![](media/de907201d91649d712e91c0b54d2f7d3.png)

### Compound Collider

For the compound collider, use a *Collider Authoring* and set the *Collider
Type* to *Compound*.

A compound collider constructs itself from children sphere and capsule
colliders. If you would like to use all children sphere and capsule colliders,
check the *Generate From Children* box. Otherwise, populate the Colliders list
with the subset of children colliders you wish to add.

![](media/75544b0cdbb419c4933bf98c8c72f543.png)

You can also generate a compound collider by attaching multiple Sphere, Capsule,
and/or Box Collider components to a single Game Object.

## Collider Types in Code

Colliders in code are all struct types which may live in any type of memory,
including stack memory. All of them may be constructed directly at any time and
in any context. However, some of these colliders may have a more complex object
such as a `BlobAssetReference` as a field.

### Collider : IComponentData

`Collider` is a union of all other collider types and serves the purpose of
representing any type of collider in an abstract matter. Its size is 48 bytes
(same as `WorldTransform`). It is the only collider type that is also an
`IComponentData`.

A default-constructed `Collider` is interpreted as a `SphereCollider` with a
`center` of (0, 0, 0) and a `radius` of 0.

`Collider`s cannot be modified directly. Instead, their values are obtained
through implicit assignment of one of the other collider types.

```csharp
Collider collider = new SphereCollider(float3.zero, 1f);
```

A `Collider` can be implicitly casted to its specialized collider type. However,
implicitly casting to the wrong type will throw an exception when safety checks
are enabled and otherwise produce undefined behavior.

To avoid this, you can check the type of a `Collider` using its `type` property.

```csharp
void TranslateColliderInColliderSpace(ref Collider collider, float3 translation)
{
    if (collider.type == ColliderType.Sphere)
    {
        SphereCollider sphere = collider;
        sphere.center += translation;
        collider = sphere;
    }
    else if (collider.type == ColliderType.Capsule)
    {
        CapsuleCollider capsule = collider;
        capsule.pointA += translation;
        capsule.pointB += translation;
        collider = capsule;
    }
}
```

### Sphere Collider

A `SphereCollider` is a struct which contains a `float3 center` and a `float
radius`, both of which are public fields. It also contains a stretch mode which
dictates how it reacts to stretched transforms. The default displaces the
sphere’s center point.

### Capsule Collider

A `CapsuleCollider` is a struct which defines the shape of a capsule using an
inner segment and a radius around that segment. The segment points are specified
by the public `float3` fields `pointA` and `pointB`. The `radius` is a public
`float` field.

By this definition, a capsule collider may be oriented along any arbitrary axis
and is not limited to the X, Y, or Z axes. Its full height can be calculated by
the following expression:

```csharp
float height = math.distance(capsule.pointA, capsule.pointB) + 2f * capsule.radius;
```

The capsule also contains a stretch mode which dictates how it reacts to
stretched transforms. The default displaces both interior points, stretching the
cylindrical part of the capsule and potentially changing the capsule’s
orientation.

### Box Collider

A `BoxCollider` is a struct which defines an axis-aligned box in local space. It
contains a `float3 center`, and a `float3 halfWidth`, which is the distances
from the center to either face along each axis. The box collider reacts to
stretched transforms exactly.

### Triangle Collider

A `TriangleCollider` is a struct which defines a single triangle in local space.
It contains 3 `float3` vertices. This type does not have a dedicated authoring
workflow and must be constructed through code. It is also used as a sub-collider
type in TriMesh Colliders.

### Convex Collider

A `ConvexCollider` is a struct which defines an immutable convex hull of up to
252 vertices.

The core of a `ConvexCollider` is its `public
BlobAssetReference<ConvexColliderBlob> convexColliderBlob `field. It is
constructed using the Unity.Physics convex hull algorithm, but with the bevel
radius disabled such that corners and edges are sharp. It can be created via
baking a *Mesh Collider* component or via a Smart Blobber.

A `ConvexCollider` exposes a `public float3 scale` which can apply a non-uniform
scale to the collider. And it reacts to stretched transforms exactly.

### TriMesh Collider

A `TriMeshCollider` is a struct which defines an immutable collection of
spatially-mapped triangles.

The core of a `TriMeshCollider` is its `public
BlobAssetReference<TriMeshColliderBlob> triMeshColliderBlob` field. Currently,
it uses the same spatial mapping structure as `CollisionLayer`, but this is
subject to change in the future. It can be created via baking a *Mesh Collider*
component or via a Smart Blobber.

A `TriMeshCollider` exposes a public `float3` scale which can apply a
non-uniform scale to the collider. And it reacts to stretched transforms
exactly.

A TriMesh Collider is a composite collider type and contains multiple
sub-colliders (in this case, all Triangles).

### Compound Collider

A `CompoundCollider` is a struct which defines a rigid immutable collection of
sphere, capsule, and box colliders and their relative transforms. Its purpose is
to allow multiple colliders to be treated as a single collider for simplicity.

The core of a `CompoundCollider` is its `public
BlobAssetReference<CompoundColliderBlob> compoundColliderBlob` field. A
`CompoundColliderBlob` contains a `BlobArray` of `Collider`s, a `BlobArray` of
`RigidTransform`s with indices corresponding to those of the `Collider`s, and an
`Aabb` which encompasses the full set of colliders in local space. In most
cases, you will never need to read this data directly.

There are three ways to create a `CompoundColliderBlob`:

1.  Use a *Collider Authoring* component
2.  Attach multiple primitive collider components to a Game Object
3.  Use a Smart Blobber from within a `Baker`

A `CompoundCollider` also exposes a `public float scale` which is a uniform
scale factor to be applied to the collider. This scale factor not only scales
the collider sizes but also their relative offsets to each other. It also
contains a `public float3 stretch` which supports transform stretching. By
default, the `stretch` vector will be rotated into each sub-collider’s local
space and then applied to that sub-collider.

A Compound Collider is a composite collider type and contains several
sub-colliders.

### Terrain Collider

A `TerrainCollider` is a struct which defines a heightmap terrain composed of
signed 16-bit integers, where four adjacent height values (quads) define two
triangles for collision. Bitmasks specify that some triangles may not be valid,
and each quad can define its triangle split direction as top left to bottom
right, or bottom left to top right.

The core of a `TerrainCollider` is its public
`BlobAssetReference<TerrainColliderBlob> terrainColliderBlob` field. A
`TerrainColliderBlob` contains the height values, hole masks, dimension
information, and triangle split orientations. It also has a hierarchical data
structure for 8-by-8 quads.

There is no built-in `Baker<UnityEngine.TerrainCollider>`. If you want a
`TerrainCollider`, you will need to bake it yourself either with the smart
blobber API provided, or via the direct blob baking API.

A `TerrainCollider` exposes a public `float3` scale which can apply a
non-uniform scale to the collider. And it reacts to stretched transforms
exactly. It also exposes a `baseHeightOffset` integer which biases the short
height values before scaling.

A Terrain Collider is a composite collider type and contains multiple
sub-colliders (in this case, all Triangles).
