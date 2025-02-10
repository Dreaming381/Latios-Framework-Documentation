# How Do I Choose Between Hybrid, Shared, and Managed Components for My Data?

Unity exposes lots of component types. And the Latios Framework adds its own to
the mix. This can be confusing for new users to pick the right one. This page
aims to answer those questions by going through each data storage type in
Entities and the Latios Framework.

## Struct IComponentData

Use this type for small, mutable, unmanaged data.

This should be your default workhorse, as it is compatible with Burst and
provides the mostly friendly memory access patterns. In general, try to keep the
size of one of these components under 128 bytes. But a few exceptions can be
fine in limited quantities.

## Class IComponentData

Use this extremely sparingly for managed types which need to be serialized on
entities and for some reason can’t be assets. For all other scenarios where you
need to serialize managed types, consider making a ScriptableObject and use
UnityObjectRef\<\> to serialize it in a struct IComponentData.

These instances aren’t pooled in any way, so you will have GC problems. Plus
they may not operate in the way you would expect references to work, as
serialization can sometime clone a new instance and sometimes share an instance
depending on the context. I think there’s a good chance these will disappear in
a future Entities version.

These are one of the two types of “Component Objects”.

## Struct IBufferElementData

Use this when you need multiple mutable instances of unmanaged data on an Entity
where all instances for a given entity can be processed by a single thread.

Dynamic Buffers are less efficient to iterate than struct `IComponentData` and
offer less flexibility than `ICollectionComponent` for splitting elements across
a parallel job. However, they are fast when the entire buffer only needs to be
touched by a single thread. They are especially powerful in filtered operations
such as when used in change filters or within an `IFindPairsProcessor`. Their
greatest power is that multiple instances can be touched within a single job.

Always set the `[InternalBufferCapacity]` for them to either a very small value
or zero. You should definitely have these in your project, but understand they
are no silver bullet.

## Struct ISharedComponentData

Use this when you want to group entities into chunks by a specific value.

The name can be a little misleading. This isn’t a mechanism for entities to
reference the same value. It is a mechanism to group entities which have the
same value. If you aren’t grouping things for query filters or to solve specific
ordering issues like the Entities Graphics’ draw filters, don’t use them.

Also, if you are able to make the contents inside the struct unmanaged, do so.
This will allow you to read the shared component values directly in Bursted
jobs, which makes them far more useful.

You may or may not choose to define one of these in your projects. They are not
essential, but they can sometimes be a great optimization tool in the right
circumstance.

## Blob Asset

Use this for complex large immutable data.

Blob Assets are immutable, so the data they contain is usually authored in the
editor or generated in external applications. While they can be used to share
data across multiple entities to reduce memory usage, doing so for
`IComponentData` under 128 bytes in size is typically not worth the effort and
may have a detrimental performance impact. At worst, they have the same access
penalties as dynamic buffers. But if you have only a few instances shared by
many entities, their accesses become cheaper. Also, many people struggle to
understand the rules behind blob asset creation and serialization. Runtime
generation of Blob Assets is currently very unsafe.

With that said, when applied to the right applications, Blob Assets are very
powerful. The Latios Framework utilizes them for several applications and hides
all the baking intricacies so you don’t have to worry about them. You’ll likely
make use of them when using the Latios Framework. It is more likely you will
find a need for them compared to `ISharedComponentData`, but it may be you won’t
ever have to write your own.

If you do need help generating your own blob assets, definitely check out the
[Smart Blobber](Smart%20Blobbers.md) documentation!

## ICleanup {ComponentData / SharedComponentData / BufferElementData}

Use these when you need to detect an entity is destroyed or want to prevent the
component’s value from being copied when instantiating new entities.

`ICleanup*` types are weird. They are powerful, but also seem to be in conflict
with the direction Unity is steering (although I doubt they will be deprecated).
Their purpose is for runtime use, where a system can identify that an entity is
destroyed and still access whatever is stored in the `ICleanup*` (if any) for
cleanup purposes. Myri uses this to detect destroyed listeners and detach them
from the DSPGraph.

Another interesting characteristic is that the component type is not added to
instantiated entities even if it exists on the source entity. In Kinemation, a
skeleton internally keeps track of all meshes bound to it for performance
reasons. But if a skeleton is instantiated, that list of dependents should not
be copied, otherwise the meshes would be bound to two skeletons at once. Even
though nothing really cares once the skeleton is destroyed, the type still uses
`ICleanup*` to achieve the desired behavior.

## Struct IManagedStructComponent – Latios Framework

Use this to store references to managed runtime resources on entities.

For the most part, if you have the ability to use `UnityObjectRef<>` instead,
you should. There are however a few reasons you might want to use an
`IManagedStructComponent`. A big reason is that they are structs, so they don’t
allocate GC perpetually as entities are instantiated and destroyed. This makes
them slightly better at referencing non-`UnityEngine.Object` types. They are
also slightly faster at accessing managed data than resolving a
`UnityObjectRef<>`. While there are situations where they can provide an
advantage, most projects will never need them.

## Struct ICollectionComponent – Latios Framework

Use this when a NativeContainer is accessed by multiple systems or when
NativeContainers need to be associated with entities.

`ICollectionComponent` when paired with blackboard entities is the Latios
Framework alternative to Unity’s singleton entities. However, you can also use
these to associate collections with any number of entities. Each instance keeps
track of the containers’ `JobHandle`s for automatic job dependency management.
If your NativeContainer is not temporary nor private to the creating system, it
should probably be in a `ICollectionComponent`. If you are not convinced, watch
the Overwatch Team’s presentation on ECS and keep an ear out for the “Replay
System Woes”.

Most of the time, you will want to attach these to a blackboard entity. But
sometimes you may prefer to use these per entity instead of a `DynamicBuffer`.
One reason you would want to do that is if you want to process each entity’s
container in a parallel job. While it is possible to do this by storing
containers in regular `IComponentData`, `ICollectionComponent` handles entity
and world lifetimes correctly and won’t leak. And that happens to be the second
reason. The lifetime management makes them excellent at storing containers other
than lists on entities if you run most of your game logic on the main thread
using idiomatic foreach.

If you subscribe to blackboard entities, definitely use these with them. As for
other entities, that can be a little more niche. You’ll probably find their best
use cases go hand-in-hand with shared components.
