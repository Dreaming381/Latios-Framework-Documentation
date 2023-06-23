# Collection and Managed Struct Components

Collection components and managed struct components are special types which can
be “attached” to entities just like any other component. They can contain native
containers and reference types respectively but can only be accessed from the
main thread.

**Due to Unity API limitations, these components are not real components and are
not part of the archetype. Instead, they “follow” an** `ExistComponent`
**generated by source generators.**

**These components are runtime-only and are not supported in Baking.**

## Creating Component Types

Creating a custom collection component or managed struct component requires
first declaring a struct that implements one of the following interfaces:
`ICollectionComponent` or `IManagedStructComponent`.

The next step is to create the fields of the struct. These are created just like
any normal struct.

The third step is to specify the struct as `partial`, so that source generators
will be able to fill in the missing pieces.

For `IManagedStructComponent` types, your custom component is ready to go. But
for `ICollectionComponent` types, there’s still one final step. You must
implement the `JobHandle TryDispose(JobHandle inputDeps)` method.

Examples:

```csharp
public struct PlanetGenerationWorld : IManagedStructComponent
{
    public LatiosWorld world;
}

public struct Pipe : IComponentData
{
    public float timeUntilNextEmission;
}

public struct PipeEmissionQueue : ICollectionComponent
{
    public NativeQueue<Entity> disabledEntityQueue;
    public NativeList<Entity>  entitiesToEnable;

    public JobHandle TryDispose(JobHandle inputDeps)
    {
        if (!disabledEntityQueue.IsCreated)
            return inputDeps;

        return JobHandle.CombineDependencies(disabledEntityQueue.Dispose(inputDeps), entitiesToEnable.Dispose(inputDeps));
    }
}
```

## Component Lifecycles

There are two ways to affect the lifecycle of an instance of a
`ICollectionComponent` or `IManagedComponent`.

### Direct Mode – LatiosWorldUnamanaged and BlackboardEntity

In direct mode, you operate on the `ICollectionComponent` and
`IManagedStructComponent` types directly. Removing a collection component will
invoke `TryDispose()` on the stored collection component.

When using the direct mode API, the contents of a collection component may or
may not be allocated. You must check for this in the `TryDispose()` method.

You can find the add and remove methods on the `LatiosWorldUnmanaged` or
`BlackboardEntity`. You may also find them as extensions to `EntityManager`, but
these extensions require an additional lookup and are consequently less
performant.

### Indirect Mode – ExistComponent

Sometimes you need to use an `EntityCommandBuffer` to instantiate entities or
add components that you wish to have a collection component. Or sometimes you
want to author a component that needs a collection component at runtime. For
these cases, you can rely on the `ExistComponent` to add or remove the
components you need. The `ExistComponent` is a nested struct type inside of your
component that is generated automatically by source generators.

When adding one of these components indirectly via the `ExistComponent`, the
component will be default-initialized. For collection components, this means
none of its Native Containers will be allocated.

When removing the `ExistComponent` to an entity, a collection component will not
have its `TryDispose()` called immediately. Instead, it will be disposed on the
next frame. The exact timing of this will be after the `SceneManagerSystem` and
`MergeBlackboardsSystem` but before other custom systems in
`LatiosInitializationSystemGroup`. The system which performs this is called
`CollectionComponentsReactiveSystem`. Other lifecycle tricks like this also
exist, but are hidden from the user.

## Getting and Setting Components

As collection components and managed struct components are not real components,
they must be fetched indirectly by their Entity. To query for these components,
query for the nested `ExistComponent` of the specific component type you are
looking for. This is a zero-sized component, and it is safe to query it as
read-only, even if you intend to write the collection component or managed
struct component.

Collection components and managed struct components can be fetched or set
through `LatiosWorldUnmanaged` methods or using a `BlackboardEntity` like the
`sceneBlackboardEntity` or `worldBlackboardEntity`. In rare situations, they can
also be accessed via `EntityManager` extension methods, but these methods are
less performant.

The following are API methods exposed for manipulating these components:

-   Managed Components
    -   AddManagedStructComponent\<T\> - Adds the managed struct component to
        the entity
    -   RemoveManagedStructComponent\<T\> - Removes the managed struct component
        from the entity
    -   GetManagedStructComponent\<T\> - Gets a copy of the managed struct
        component from the entity, copying references to managed types rather
        than their underlying objects
    -   SetManagedStructComponent\<T\> - Replaces the stored managed struct
        component with a copy of the passed in struct, copying references to
        managed types
    -   HasManagedStructComponent\<T\> - Checks if a stored managed struct
        component exists on the entity and is not pending removal
-   Collection Components
    -   AddOrSetCollectionComponentAndDisposeOld\<T\> - Adds the collection
        component to the entity, or replaces an existing one if one is already
        added and disposes the previous instance
    -   RemoveCollectionComponentAndDispose\<T\> - Removes the collection
        component and disposes it
    -   GetCollectionComponent\<T\> - Gets the collection component from the
        entity and marks it as read-only if `readOnly` is manually set to `true`
    -   SetCollectionComponentAndDisposeOld\<T\> - Replaces the stored
        collection component from the entity with the passed in collection
        component and disposes the replaced instance
    -   HasCollectionComponent\<T\> - Checks if a stored collection component
        exists on the entity and is not pending removal

## Collection Component Dependency Management

Collection components have an intrinsic understanding of the `Dependency`
property of systems, similar to lambda jobs. This means that **by default**,
dependency management is **automatic**!

*Note: This only works if the system is updated via a Latios Framework system
updater. The default system groups and all Latios Framework ComponentSystemGroup
types will handle this correctly.*

### ReadOnly and ReadWrite per Instance

The automatic dependency management tracks separate `ReadOnly` and `ReadWrite`
`JobHandles` per instance of a collection component. The rules for this follow
the same rules as `ComponentType`s in Unity’s ECS:

-   Multiple independent jobs may read from a collection component
    simultaneously
-   When a job has write-access to a collection component, no other job may read
    nor write to that collection component simultaneously

The automatic dependency management will do its best to keep track of this for
you, but requires that you correctly specify whether you are accessing a
collection component as `readOnly` similar to requesting a `ComponentLookup`.

*Note: You specify read-only status when requesting the actual collection
component value, not when querying the* `ExistComponent`*.*

However, unlike Unity’s ECS, each instance is tracked per entity. Taking from
the example above, this means a job writing to `EntityA`’s `PipeEmissionQueue`
can run simultaneously with a job writing to `EntityB`’s `PipeEmissionQueue`.

### How the Dependencies are Updated

Before the `OnUpdate` of the system executes, the system dispatcher registers it
with the `LatiosWorld` as the active running system. From there, the
`LatiosWorld` forwards all `Dependency` updates to that system and also records
a list of all `Entity`-`ICollectionComponent` pairs which have been accessed and
need their internal `JobHandle`s updated. After the `OnUpdate` finishes, the
system dispatcher passes the final `Dependency` to the `LatiosWorld` which then
commits the `Dependency` to the internal storage.

**Warning:** Be careful when iterating through collection components in an
`Entities.ForEach` loop like the following:

```csharp
Entities.WithAll<FactionTag>().ForEach((Entity entity) =>
{
    var shipLayer = EntityManager.GetCollectionComponent<FactionShipsCollisionLayer>(entity, true).layer;
    Dependency = Physics.FindPairs(bulletLayer, shipLayer, processor).ScheduleParallel(Dependency);
}).WithoutBurst().Run();
```

This will complete Dependency at the start of the `Entities.ForEach`, which may
include jobs for components you may be trying to run new jobs on inside your
loop that interact with your collection components.

To avoid such a sync point, you can do something like this:

```csharp
var backup = Dependency;
Dependency = default;

Entities.WithAll<FactionTag>().ForEach((Entity entity, int entityInQueryIndex) =>
{
    if (entityInQueryIndex == 0)
        Dependency = backup;

    var shipLayer = EntityManager.GetCollectionComponent<FactionShipsCollisionLayer>(entity, true).layer;

    Dependency = Physics.FindPairs(bulletLayer, shipLayer, processor).ScheduleParallel(Dependency);
}).WithoutBurst().Run();
```

API to handle this scenario more conveniently may come in a future release.

### Fine-Grained Dependency Control

If for some reason, you wish to have more fine-grained control over the
dependency management, the following table shows which functions modify which
`JobHandle` values:

| Method                                         | Modifies Dependency | Queues internal update from Dependency | Removes internal update from queue |
|------------------------------------------------|---------------------|----------------------------------------|------------------------------------|
| AddOrSetCollectionComponentAndDisposeOld\<T\>  |                     | x                                      |                                    |
| RemoveCollectionComponentAndDispose\<T\>       | x                   | x                                      |                                    |
| GetCollectionComponent\<T\>                    | x                   | x                                      |                                    |
| GetCollectionComponent\<T\> out JobHandle      |                     | x                                      |                                    |
| SetCollectionComponentAndDisposeOld\<T\>       |                     | x                                      |                                    |
| UpdateCollectionComponentDependency\<T\>       |                     |                                        | x                                  |
| UpdateCollectionComponentMainThreadAccess\<T\> |                     |                                        | x                                  |

One reason why you may wish to manually control dependencies is to allow jobs to
run during a sync point. By scheduling a job which only interacts with
collection components attached to `worldBlackboardEntity` and calling
`UpdateCollectionComponentDependency<T>()` on those collection components while
not touching `Dependency`, the `JobHandle`s will be automatically managed by the
Latios Framework’s automatic dependency management system but not Unity ECS’s
automatic dependency management system.