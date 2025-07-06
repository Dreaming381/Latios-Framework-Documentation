# LifeFX

LifeFX is a visual effects and graphics programming module built on top of
Kinemation. Currently, it mainly serves as a bridge to Unity VFX Graph. But this
bridge can also be leveraged for many other graphics operations.

Check out the [Getting Started](Getting%20Started.md) page!

## Features

### Easy GPU Events

LifeFX makes it easy to send GPU event structures to the GPU using its *postal
system*. Events of the same type are batched together into a single
`GraphicsBuffer`. Ranges within this buffer can be broadcast to various
consumers. `ScriptableObjects` are used as a serialized mechanism for mapping
events to the appropriate consumers. At runtime, the `ScriptableObjects` are
used in `UnityObjectRef` form, allowing event creation to be performed entirely
in Burst-compiled jobs.

### Easy Transform Tracking

LifeFX has a built-in transform tracker that automatically syncs QVVS Transforms
to the GPU at stable buffer indices. A GPU event can be sent after an entity is
spawned with the tracking index, and then the GPU can read from that index in
subsequent frames. 2 bits of the QVVS worldIndex are used to specify alive and
enabled states, so the GPU can recognize when the entity is destroyed or has
tracking temporarily disabled.

### GameObject Broadcast System

LifeFX makes it easy for GPU events and other graphics buffer data generated
from ECS to be broadcasted to the world of Game Objects via Transform’s
GameObjectEntity feature. This simplifies integration with custom graphics
assets which are able to receive data via graphics buffers.

### VFX Graph Ready

LifeFX comes with out-of-the-box drivers to forward broadcasted graphics buffers
to VFX Graph. This setup allows a single VFX Graph instance to service thousands
of entities, with real-time graph editing in play mode. This replicates the
design philosophy seen in Unity’s Galaxy Sample, except it is more modular and
easier to set up.

In addition, LifeFX provides VFX Subgraphs as samples for working with tracked
transforms as QVVS Transforms.

## Known Issues

-   Some of the VFX Subgraphs included do not work due to bugs in Unity’s VFX
    Graph parser. Please help report bugs to Unity when you encounter them. And
    if you find workarounds, please share them with the community.

## Near-Term Roadmap

-   High Fidelity Mesh Cloth
-   VFX Graph Sample Effects
