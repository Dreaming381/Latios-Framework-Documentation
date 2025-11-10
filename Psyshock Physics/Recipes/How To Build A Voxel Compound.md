# How To Build A Voxel Compound

If you are making a voxel-based game involving rigid bodies, you may need to
create compound colliders at runtime with mass distributions that behave
realistically. This guide will walk you through how to do that.

## Prerequisites

To accomplish this, you’ll need two things.

1.  The raw voxel positions and material densities of each voxel
2.  Box colliders representing the combined voxel colliders

The box colliders are intended to represent a simplified surface for collision.
Fewer box collider shapes will result in faster processing.

## The Strategy

We’ll be making use of a sequence of Psyshock APIs to complete this process:

1.  Find the center of mass using `TrueSim.CenterOfMassFrom()`
2.  Calculate each voxel’s gyration tensor diagonal using
    `TrueSim.DiagonalizedGyrationTensorOfBox()`
3.  Convert each diagonal to a matrix using `TrueSim.InertiaTensorFrom()`
4.  Translate each matrix relative to the center of mass using
    `TrueSim.TranslateInertiaTensor()`
5.  Compose the compound gyration tensor matrix from
    `TrueSim.GyrationTensorFrom()`
6.  Build the compound using `CompoundColliderBlob.BuildBlob()`

## The Details

For simplicity, let’s define this struct as our voxel.

```csharp
struct Voxel
{
    public float3 position;
    public float mass;
}
```

To get the center of mass, we’ll need to decompose these fields into separate
spans. Then we pass those spans into `TrueSim.CenterOfMassFrom()`.

```csharp
System.Span<float3> voxelCenters = stackalloc float3[voxels.Length];
System.Span<float>  voxelMasses  = stackalloc float[voxels.Length];
for (int i = 0; i < voxels.Length; i++)
{
    voxelCenters[i] = voxels[i].position;
    voxelMasses[i]  = voxels[i].mass;
}
var centerOfMass = TrueSim.CenterOfMassFrom(voxelCenters, voxelMasses);
```

Next, we want to build the gyration tensor of each voxel relative to the center
of mass. Remember that the gyration tensor is simply a unit mass inertia tensor,
so most APIs that operate on inertia tensors can also be used on gyration
tensors.

At this point, we’ll need to know the dimensions of each voxel. We’ll assume
each voxel is 1x1x1 units, which means the distance from the center to each side
is `0.5f`.

None of our voxels are rotated in the compound’s local space, so we can use an
identity orientation quaternion when asked for one.

```csharp
System.Span<float3x3> gyrationTensors = stackalloc float3x3[voxels.Length];
for (int i = 0; i < voxels.Length; i++)
{
    var diagonal                 = TrueSim.DiagonalizedGyrationTensorOfBox(new float3(0.5f));
    var voxelLocalGyrationTensor = TrueSim.InertiaTensorFrom(quaternion.identity, diagonal);
    gyrationTensors[i]           = TrueSim.TranslateInertiaTensor(voxelLocalGyrationTensor, voxelCenters[i] - centerOfMass);
}
```

Lastly, we can combine all the gyration tensors into one, and then use
everything we have to build the compound collider.

```csharp
var compoundGyrationTensor = TrueSim.GyrationTensorFrom(gyrationTensors, voxelMasses);
var compoundBlob           = CompoundColliderBlob.BuildBlob(ref blobBuilder, boxColliders, centerOfMass, compoundGyrationTensor, Allocator.Persistent);
```
