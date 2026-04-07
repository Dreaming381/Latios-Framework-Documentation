# Getting Started with Aux ECS

Aux ECS is a convenient storage solution when you need a complex data structure
that tracks multiple entities. It is best to reason about it as a collection,
and less of an architecture that you might find in more feature-rich ECS
solutions such as Unity’s.

## Creating an AuxWorld

The center of Aux ECS is the `AuxWorld`. The `AuxWorld` serves as the container
of all components and entity associations. As a Unity ECS analogy, `AuxWorld`
additionally functions as the `EntityManager`. To create an `AuxWorld`, simply
call its constructor, and pass in an allocator. The allocator will be used for
storing all attached components, as well as holding all associations.

Typically, you’d use a persistent allocator, but you can also use a temporary
allocator if you need to mark up a bunch of entities in a single frame. Just
remember that if you don’t explicitly dispose an `AuxWorld`, any
`IAuxDisposable` components will not have `Dispose()` called. More on those
later.

## Adding, Modifying, and Removing Components

To add a component to an entity, simply call `AuxWorld.AddComponent<T>()` and
pass in the `Entity` and any unmanaged struct you wish to attach. Aux ECS
assumes that every possible combination of Index and Version is a valid entity
that by default has no components attached until one is added. There is no such
thing as destroying an entity. Instead, you can remove individual components
with `AuxWorld.RemoveComponent<T>()`. You can also remove all components
currently attached by calling `AuxWorld.RemoveAllComponents()`.

To retrieve a component from a specific entity, simply call
`AuxWorld.TryGetComponent<T>()`. This returns true if the entity has the
component. The `out` parameter is an `AuxRef<T>` which is a safe reference to
the component. With an `AuxRef<T>`, you can access the underlying reference via
the `aux` property.

The `AuxRef<T>` is only invalidated when the component is removed from the
entity entirely. If the component is replaced via `AddComponent<T>()`, then the
reference will refer to the new value automatically.

## IAuxDisposable

Aux ECS allows you to store structs that have full containers inside of them,
replacing the need for `DynamicBuffer` or other similar constructs. However,
these structures still need to be cleaned up when the component is removed or
the `AuxWorld` is disposed. You can implement the `IAuxDisposable` interface for
this.

`IAuxDisposable` uses the `VPtr` source generator feature of Core. To ensure
your component is compatible, mark the struct as partial, and ensure it also
implements the `IVInterface` interface alongside `IAuxDisposable`. The following
is an example of what the declaration should look like:

```csharp
partial struct ExampleDisposable : IAuxDisposable, IVInterface
{
    UnsafeList<int> intList;

    public ExampleDisposable(AuxWorld auxWorld)
    {
        // Allocate with the AuxWorld allocator for extra safety.
        // If the AuxWorld was allocated with a temporary allocator and never disposed,
        // ExampleDisposable.Dispose() won't be called, but that's fine if it is using
        // the same temporary allocator.
        intList = new UnsafeList<int>(32, auxWorld.allocator);
    }

    public void Dispose()
    {
        intList.Dispose();
    }
}
```

`IAuxDisposable` is automatically detected via `AuxWorld.AddComponent<T>()`. You
don’t have to do anything special. `Dispose()` is automatically called under
these circumstances:

-   `AuxWorld.RemoveComponent<T>()` is called specifying an `IAuxDisposable`
    type
-   `AuxWorld.RemoveAllComponents()` is called which invokes `Dispose()` for
    each `IAuxDisposable` component attached to the entity
-   `AuxWorld.AddComponent<T>()` is called specifying an `IAuxDisposable` type
    that is already present on the entity, in which the old value will have
    `Dispose()` called
-   `AuxWorld.Dispose()` calls `Dispose()` on every `IAuxDisposable` stored in
    the `AuxWorld`.

## Querying Just One Component

If you only need to update an individual component type across all entities, you
can use `AuxWorld.AllOf<T>` to query for that type. This provides an enumerator
of `AuxRef<T>` which you can use in a foreach expression.

The following shows an example of iterating all components of type `float3`.

```csharp
foreach (var float3Ref in auxWorld.AllOf<float3>())
{
    float3Ref.aux.x += 1f;
}
```

You probably want to define your own struct types rather than directly add
`float3` as a component on your entities. But this shows that Aux ECS really
allows you to add any unmanaged type.

`AllOf<T>` iterates components in their stored order, which makes this method
very cache-friendly. Prefer to use it when it meets your needs.

## Querying Tuples

If you need to access multiple components, or perhaps need to access the Entity
instances the components are attached to, then you can use `AuxWorld.AllWith<T0,
…>()`. This provides an enumerator to a `Tuple`, which contains the `Entity` and
an `AuxRef` for each queried component. `Tuple` can be deconstructed, and when
doing so, the `Entity` is always the first element, followed by each queried
component.

The following shows an example of iterating a tuple of `float3` and `int`.
Again, you probably should use your own types as components.

```csharp
foreach ((var entity, var float3Ref, var intRef) in auxWorld.AllWith<float3, int>())
{
    UnityEngine.Debug.Log($"{entity.ToFixedString()} has float3: {float3Ref.aux} and int: {intRef.aux}");
}
```

## Aux Ref Life-Cycle

The `AuxRef` type remains valid as long as the entity has the type of component
attached. It is invalidated when it is removed. If the type is re-added after
removal, the `AuxRef` will remain invalid.

However, when calling `AddComponent<T>()` on an entity which already contains an
existing component of type `T`, then the new value will overwrite the old value
at the same memory address, thus preserving the validity of `AuxRef`.

Because `AuxRef` is difficult to invalidate, it is often a good idea to store it
inside of other components in the same `AuxWorld` for faster lookups. A safety
check in the Editor will catch when you try to access an invalid AuxRef (except
if the AuxWorld is disposed prematurely), which means this pattern can be
debugged if you make mistakes.

## Container Safety

`AuxWorld` is simply a pointer to its underlying container, and this has
consequences to safety. Let’s suppose you decided to store an `AuxWorld` inside
a `NativeReference`. Because `AuxWorld` is just a pointer, you don’t have to
change its value to add or remove components. And consequently, if you added a
`[ReadOnly]` to the `NativeReference` in a job, the job system will be unaware
of writing operations you perform. For this reason, **never use** `[ReadOnly]`
**with anything containing AuxWorld!**

`AuxWorld` is really intended to be used with full write-access either on the
main thread, the audio thread, a custom thread, or in a single-threaded job.
