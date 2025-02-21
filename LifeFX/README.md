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

## Known Issues

-   Entity tracking (updating positions for things like trails) is not provided
    out-of-the-box. There isn’t a one-size-fits-all optimal way to do this.
    Solutions for common use cases may be provided in the Add-Ons package.

## Near-Term Roadmap

-   VFX Graph Nodes and Templates
