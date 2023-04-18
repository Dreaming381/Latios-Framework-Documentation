# Dynamic Meshes

Dynamic Meshes are meshes that leverage Kinemation’s GPU deform buffers to feed
in mesh modifications from ECS data. They can be used to represent non-rigid
shapes or for CPU-based particle systems.

Dynamic meshes reupload their mesh every frame in which they are visible and
rendered, even if no changes have been made. They maintain their own motion
history via an internal buffer rotation mechanism.

## Composition

A Dynamic Mesh requires four components to operate correctly:

-   MeshDeformDataBlobReference : IComponentData
-   DynamicMeshMaxVertexDisplacement : IComponentData
-   DynamicMeshState : IComponentData
-   DynamicMeshVertex : IBufferElementData

There is also a `DynamicMeshAspect : IAspect` which abstracts `DynamicMeshState`
and `DynamicMeshVertex`.

Lastly, to render a Dynamic Mesh correctly, the entity will need to be baked as
a *deformed mesh*.

## Baking a Dynamic Mesh

First, create an authoring component that inherits OverrideMeshRendererBase.
This action will disable default Mesh Renderer baking.

```csharp
public struct TestDynamicMeshData : IComponentData
{
    public float2 meshRelativeCenter;
    public float  frequency;
    public float  speed;
    public float  amplitude;
}

public class TestDynamicMeshAuthoring : OverrideMeshRendererBase
{
    public float2 meshRelativeCenter;
    public float  frequency;
    public float  speed;
    public float  amplitude;
}
```

Next, create a smart baker for your authoring component. Along with any custom
data you require, this baker must invoke `BakeDeformMeshAndMaterial()`, and
request a smart blobber to generate the `MeshDeformDataBlob` required by the
`MeshDeformDataBlobReference` component. It must also create the remaining
required runtime components.

```csharp
[TemporaryBakingType]
struct TestDynamicMeshBakeItem : ISmartBakeItem<TestDynamicMeshAuthoring>
{
    SmartBlobberHandle<MeshDeformDataBlob> meshBlobRequest;

    public bool Bake(TestDynamicMeshAuthoring authoring, IBaker baker)
    {
        baker.AddComponent(new TestDynamicMeshData
        {
            meshRelativeCenter = authoring.meshRelativeCenter,
            frequency          = authoring.frequency,
            speed              = authoring.speed,
            amplitude          = authoring.amplitude,
        });

        // Typically, you'd want to cache this in the SmartBaker or a static and reuse it.
        List<Material> materials = new List<Material>();

        var renderer = baker.GetComponent<MeshRenderer>();
        var mesh     = baker.GetComponent<MeshFilter>().sharedMesh;
        renderer.GetSharedMaterials(materials);
        baker.BakeDeformMeshAndMaterial(renderer, mesh, materials);

        meshBlobRequest = baker.RequestCreateBlobAsset(mesh);
        baker.AddComponent<MeshDeformDataBlobReference>();
        baker.AddComponent<DynamicMeshMaxVertexDisplacement>();
        baker.AddComponent(DynamicMeshAspect.RequiredComponentTypeSet);

        return true;
    }

    public void PostProcessBlobRequests(EntityManager entityManager, Entity entity)
    {
        entityManager.SetComponentData(entity, new MeshDeformDataBlobReference
        {
            blob = meshBlobRequest.Resolve(entityManager)
        });
    }
}

class TestDynamicMeshAuthoringBaker : SmartBaker<TestDynamicMeshAuthoring, TestDynamicMeshBakeItem>
{
}
```

*Q:* `DynamicBuffer<DynamicMeshVertex>` *is left empty?*

If you want to initialize the mesh data in the baker, you may do so. However, if
you leave it zero-sized, the runtime will initialize it to the contents of
`MeshDeformDataBlob.undeformedVertices`. This drastically decreases the size of
the serialized subscene.

## Modifying a Dynamic Mesh at Runtime

To write to a Dynamic Mesh, simply acquire the mesh data via
`DynamicMeshAspect.verticesRW`. These vertices will **NOT** contain the previous
frame’s vertices and should only ever be written to. If you need the previous
frame’s modifications, read from the `previousVertices` instead.

```csharp
public partial struct TestDynamicMeshSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        new Job { elapsedTime = (float)SystemAPI.Time.ElapsedTime }.ScheduleParallel();
    }

    [BurstCompile]
    partial struct Job : IJobEntity
    {
        public float elapsedTime;

        public void Execute(ref DynamicMeshAspect mesh, ref DynamicMeshMaxVertexDisplacement displacement, in TestDynamicMeshData data)
        {
            var vertices = mesh.verticesRW;
            var previous = mesh.previousVertices;
            for (int i = 0; i < mesh.vertexCount; i++)
            {
                var vertex   = previous[i];
                var distance = math.distance(vertex.position.xz, data.meshRelativeCenter);
                math.sincos(data.frequency * distance + data.speed * elapsedTime, out var s, out var c);
                vertex.position.y = s * data.amplitude;
                var slope         = c * data.amplitude;
                var normal2d      = math.normalize(new float2(-slope, 1));
                vertex.normal     = math.normalize(new float3(normal2d.x * math.normalize(vertex.position.xz - data.meshRelativeCenter), normal2d.y)).xzy;
                vertices[i]       = vertex;
            }
            displacement.maxDisplacement = data.amplitude;
        }
    }
}
```

When modifying a `DynamicMesh`, you need to compute the maximum displacement of
any vertex from its original position in the
`MeshDeformDataBlob.undeformedVertices` and store it in
`DynamicMeshMaxVertexDisplacement`, otherwise culling may not work correctly.

However, modifying the `DynamicMeshMaxVertexDisplacement` every frame has a
non-zero cost, especially if the mesh is skinned or has blend shapes. If an
upper limit is known, it may be sufficient to set the value once at runtime or
even during baking to avoid triggering the change filter every frame.

## Particle Tricks

When working with Dynamic Meshes as particles, you’ll still need a base mesh to
work from. While any mesh of sufficient size will do, you must make sure that
the triangles are set up correctly. For example, if working with billboard
quads, you would want a mesh that for every 4 vertices has 6 indices
representing those 4 vertices as 2 triangles.

### Culling Primitives

To cull a vertex, simply assign `math.asfloat(~0u)` to the `position`.

In general, you want to do this for all vertices in a primitive. In our
billboard quads case, all four vertices of a quad should have this position. VFX
Graph uses the same trick.

### Generating UVs

Dynamic Meshes do not allow dynamic assignment of UVs, as this is not included
in Kinemation’s deform buffers. This means you will either have to rely on the
UVs within the mesh or generate them on the fly in the shader using the Vertex
ID. Here’s an example of how you can construct the UVs in Shader Graph:

![](media/69b6ce5f29c5e2ca2b94e817b39128de.png)
