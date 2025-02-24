# Getting Started with LifeFX

LifeFX provides a pipeline for extracting data from ECS and sending it to GPU
buffers that can be processed by Game Objects. The most common use case for this
is to drive VFX Graph instances in a scene. This guide will discuss this
workflow for spawning multiple effect instances within a single VFX Game Object
in a scene, with the assumption you are experienced with VFX Graph. For other
use cases, it may be best to seek help from the framework’s Discord community.

## Requirements and Installation

LifeFX requires that GameObjectEntity be installed when using Unity Transforms.

LifeFX is disabled in the bootstrap templates, and must be enabled by
uncommenting its installer in the `LatiosBootstrap` class. LifeFX currently has
no baking installation requirements and does not need to be installed in the
Editor world. When LifeFX is installed, it will add
`Kinemation.EnableCustomGraphicsTag` to the `worldBlackboardEntity`, which will
enable the early dispatch systems. To understand what this means and when LifeFX
systems update, refer to [Kinemation Custom
Graphics](../Kinemation%20Animation%20and%20Rendering/Custom%20Graphics.md).

## Terminology

LifeFX has specific terminology for its data pipeline.

### Graphics Event

A graphics event is just a struct. It contains a set of data that is independent
and unordered to everything else. All instances of the same type of event are
sent to the GPU in a single structured `GraphicsBuffer`. For VFX Graph, this
event type should be one of the [supported
types](https://docs.unity3d.com/Packages/com.unity.visualeffectgraph@17.1/manual/Operator-SampleBuffer.html#available-types).

### Tunnel

A tunnel is an asset which describes a common link channel between ECS data and
Game Objects. This creates a decoupling of scenes and subscenes, and also
decouples graphics event generation from how they may be used. In ECS, tunnels
should be baked into unmanaged ECS components and buffers as `UnityObjectRef`s.
For Game Objects, tunnels are serialized directly in buffer receptors.

Tunnels are user-defined types which derive from `GraphicsEventTunnel<T>`, where
`T` is the graphics event type.

### Buffer Receptors

A buffer receptor is a Game Object with a Game Object Entity (no host required).
Each frame, it will receive an update from ECS with a `GraphicsBuffer`, as well
as a start index into the buffer and a count in the case of an event buffer.
This range of the buffer contains all the events that were broadcasted through
the tunnel the buffer receptor references. For non-event receptors, the buffer
will be some global resource such as deformation vertices.

Buffer Receptors can be used as a base class, or as a publisher via either C\#
events or Unity Actions. Multiple receptors can listen to the same tunnel or
global resource.

### VFX Graph Buffer Providers

A buffer provider is a derived class of a buffer receptor which forwards the
buffer and ranges to a VFX Graph instance. In the inspector, the user specifies
the VFX Graph shader variable names for the buffer, start, and count.

### Graphics Event Postal

The `GraphicsEventPostal` is a collection component on the
`worldBlackboardEntity` that serves as the receiver for graphics events
generated in ECS. You retrieve this component as `readOnly` and add events to
it. The reason it is `readOnly` is because you are only reading the array of
pointers to thread-safe collector bins. It is completely safe to post graphics
events from multiple jobs concurrently.

Events are read during the Collect stage of `GraphicsEventUploadSystem` within
`DispatchRoundRobinLateExtensionsSuperSystem`. `GraphicsEventUploadSystem` only
runs the first time it is invoked in a frame.

### Mailbox

A mailbox is a thread-safe collector bin for graphics events of a specific type.
You retrieve mailboxes from the `GraphicsEventPostal`. You can either retrieve a
mailbox within a job, or retrieve it on the main thread and pass it to the job.
In a mailbox, you send an event by passing it the event data and the tunnel as a
`UnityObjectRef`.

`GraphicsEventPostal` has a shorthand method for retrieving the mailbox and
writing the event in a single method for convenience.

### Shader Property to Global Buffer Map

The `ShaderPropertyToGlobalBufferMap` is a collection component on the
`worldBlackboardEntity` that associates global shader buffer variables with
their active graphics buffers. Its purpose is to broadcast global buffers that
were obtained from Kinemation’s `GraphicsBufferBroker`. Just like
`GraphicsEventPostal`, you want to retrieve this component as `readOnly`.

The map component is read during the Dispatch stage of
`GraphicsGlobalBufferBroadcastSystem` inside
`DispatchRoundRobinLateExtensionsSuperSystem`.
`GraphicsGlobalBufferBroadcastSystem` only runs the first time it is invoked in
a frame.

## Making Your First LifeFX Effect

### ECS and Tunnel

First, create a new script which will represent your tunnel type. Use the
`ScriptableObject` template as the basis. Make your new type inherit from
`GraphicsEventTunnel<SomeType>` where `SomeType` can be any type you want to
represent your graphics event. If you need to define your own type, do so now,
and be sure to add the required
`[VFXType(VFXTypeAttribute.Usage.GraphicsBuffer)]` attribute. If you want a raw
type, you can also use Unity Mathematics types instead of Vector and Matrix
types such as `float3`.

Next, create an asset instance from your new tunnel type. If you don’t know how,
your script is probably missing the `[CreateAssetMenu]` attribute.

Now, create an ECS component and authoring component for your effect, with a
reference to your tunnel. If you’d like, you can add some timing variables for
spawning logic. Here’s an example:

```csharp
struct PositionEventSpawner : IComponentData
{
    public float                                       timeUntilNextSpawn;
    public float                                       spawnPeriod;
    public UnityObjectRef<PositionGraphicsEventTunnel> tunnel;
}
```

In the above example, `PositionGraphicsEventTunnel` uses a float3 graphics event
type. We could change the type of tunnel to `GraphicsEventTunnel<float3>` if we
wanted to.

Next, create a system to grab the `GraphicsEventPostal` and spawn events. Here’s
an example for the previous spawner:

```csharp
[RequireMatchingQueriesForUpdate]
[BurstCompile]
public partial struct PositionEventSpawnerSystem : ISystem
{
    LatiosWorldUnmanaged latiosWorld;

    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        latiosWorld = state.GetLatiosWorldUnmanaged();
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var mailbox = latiosWorld.worldBlackboardEntity.GetCollectionComponent<GraphicsEventPostal>(true).GetMailbox<float3>();
        foreach ((var transform, var spawner) in SystemAPI.Query<WorldTransform, RefRW<PositionEventSpawner> >())
        {
            ref var sp             = ref spawner.ValueRW;
            sp.timeUntilNextSpawn -= SystemAPI.Time.DeltaTime;
            if (sp.timeUntilNextSpawn < 0f)
            {
                sp.timeUntilNextSpawn += sp.spawnPeriod;
                mailbox.Send(transform.position, sp.tunnel);
            }
        }
    }
}
```

The system is allowed to update anywhere before `GraphicsEventUploadSystem`.
This includes anywhere in `InitializationSystemGroup` and
`SimulationSystemGroup`. It also includes anywhere in `PresentationSystemGroup`
before `KinemationPostRenderSystemGroup`. Or it can update in
`KinemationCustomGraphicsSetupSuperSystem`.

### VFX Graph and Game Objects

We are completely done with writing code at this point.

Now it is time you create your VFX Graph. Making the graph is out-of-scope for
this guide, but here are some tips on what you are trying to do.

Most likely, you want a dedicated instancer spawn system. This is a system that
creates a single particle for each instance you want to create, and that
particle immediately dies in the Update context emitting a GPU spawn event. The
spawn event is what is responsible for the actual spawning of instances in a
second system. One way to set this up is to store the `spawnIndex` of the
instancer spawn particle in some custom attribute that can be inherited by the
particles in the second system. Then the second system can use it to sample the
graphics buffer containing the graphics events. Another approach is to have the
instancer spawn particles from the first system sample the buffer, and then
particles in the second system inherit all the relevant properties.

In any case, you will need to sample the graphics buffer using the `spawnIndex`.
You must add the graphics event start index to the `spawnIndex` and use that as
the sample index.

Once you have your graph set up, you will need a Visual Effect Game Object in
your scene (**not a subscene**). Add a VFX Graph Event Buffer Provider component
to the Game Object. Drag in your tunnel, and set the names of your graph
properties for the buffer name, start, and count. It should look something like
this:

![](media/3248cce2cd5492328abd972ade9df7a1.png)

That’s it!

You can enter play mode, and hopefully everything works! In play mode, you can
bind the graph editor to the Visual Effect in your scene and live edit it while
your ECS simulation running.

## Q&A

**Do I really need every Visual Effect instance in the scene? Can’t LifeFX load
them dynamically?**

LifeFX on the ECS side only knows the tunnel assets which were serialized and
loaded from the subscenes. It has no idea what VFX graphs are tied to which
tunnels. You can spawn the Visual Effect `GameObjects` yourself (this is fine
with GameObjectEntity as long as you don’t specify a host), but then you are
responsible for managing the lifecycle of such `GameObjects`.
