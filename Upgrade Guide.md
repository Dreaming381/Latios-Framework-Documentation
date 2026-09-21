# Upgrade Guide [0.15.12] [0.16.0]

If you are not using version control, please back up your project in a separate
directory before upgrading! You may need to reference the pre-upgraded project
during the upgrade process.

This document only highlights the common things to be aware of when upgrading.
See the changelogs of each module for a complete list of changes.

## LifeFX

### Operator Subgraph Updates

You may want to reimport the samples package to update the subgraphs if you find
they don’t work out-of-the-box. Some small changes were made that fixes the
broken subgraphs. All the subgraphs work correctly now.

## Transforms

### TransformsKey Change

`TransformsKey` used to have two validation modes: one at the chunk level, and
the other at the entity level. The chunk level was removed. Instead,
`TransformAspect` provides `TransformsKey` as a property, making it much easier
to access.

## Unika

### Cached Resolver Change

Cached resolvers had a small breaking API change. See the changelog for details.

## Core

### EntityOperationCommandBuffer

Some operations in `EntityOperationCommandBuffer` have been deprecated. They
relied upon ENTITY_STORE_V1 which is going away. The Latios Framework no longer
requires that scripting define.

## New Things to Try!

### Core

Core got some massive source generator improvements, with a completely new way
to write systems. `IInjectable` let’s you inject type handles and lookups in a
job without having to assign all of them at schedule time. And `IJobEach` is an
alternative to `IJobEntity` with a lot of really nice extra features and
ergonomics. This is all backed by `ILatiosApi`, which your systems can
implement. You no longer need to cache the `LatiosWorldUnmanaged` yourself.

Core also has the new ticking framework, for fixed rate simulation and
controlled interpolation logic. This serves as the first layer of a future
networking solution.

There’s a new custom command buffer, and it is called `CustomCommandBuffer`.
That’s right. You can create your own sync point commands. In addition, it and
`InstantiateCommandBufferCommandX` now have a dedicated allocator for allocating
arrays to store inside commands.

Lastly, `BlobVPtr` is a thing. That means you can have polymorphic blobs.

### QVVS Transforms

This module got a major update. With `IJobEach`, iterating `TransformAspect` is
easy again! Just plop it into the `Execute()` method parameters, and the
framework does the rest!

There’s also new aspect types. `TransformDeferableAspect` lets you batch up
multiple writes to a hierarchy and have them process in a single sweep.
`TransformReadAspect` gives you an easy way to read transforms and navigate the
hierarchy. Also, the `TransformsKey` can now be found on `TransformAspect` and
`TransformDeferableAspect`, which makes it easy to reason about entire
hierarchies inside a simple `IJobEach`.

### Calci

You can now use `RngEach` inside an `IJobEach` as an `Execute()` parameter, and
not have to worry about setup at all.

### Kinemation

AclUnity now has iOS plugins included! The only platform left is web, which may
come when the framework targets a newer LTS version.

`OptimizedSkeletonInputAspect` can be used inside `IJobEach` as an `Execute()`
parameter to make setting up an `OptimizedSkeletonAspect` easier.

Unique Mesh has a new mode for specifying vertices in a raw `MeshData`-like
format. You can define the attributes directly, including their data types and
strides.

### Unika

Unika finally gets to love up to its vision. It now has an automatic authoring
generator. It will really feel like a `MonoBehaviour`. Perfect for prototyping,
one-off logic, and tech artist friendliness.

Unika also has a proper means to inspect scripts in a debugger. And you can also
see script contents in a new tab within the Entity runtime inspector.

And if remembering to add types to a `ComponentBroker` was confusing, there’s
now a `[Needs]` attribute system that lets scripts declare what they need in
specific contexts, which can automatically populate the `ComponentBroker`.
