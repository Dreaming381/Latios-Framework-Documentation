# Aux ECS

Aux ECS is an auxiliary single-threaded ECS intended as a storage for
specialized features and mechanisms. This storage is unique in that rather than
creating and managing its own entities with their own handles, it uses
`Unity.Entities.Entity` as a key, without any knowledge of that entity’s
lifecycle.

Check out the [Getting Started](Getting%20Started.md) page!

## Features

### Burst Ready Storage

Create an `AuxWorld` and store it in a `NativeReference` or other collection for
easy access. `AuxWorld` is also generally safe to pass around by value, as it is
just a pointer to the underlying storage.

Once you have an `AuxWorld`, attach any unmanaged struct to any `Entity` value.
It doesn’t matter if the `Entity` is real or fake, alive or dead.

### Memory Stable

Once a component is attached to an entity, its stored memory address never
changes until it is removed. This allows components to reference each other
directly, making Aux ECS suitable for representing complex graph structures
without heavy use of table lookups. Lifecycle safety can be ensured with the
`AuxRef` wrapper (as long as the `AuxWorld` is alive).

### Easy Container Lifecycle

Components can implement the `IAuxDisposable` interface ([a VPtr
interface](../Core/VPtr.md)) to automatically clean up memory when the component
is removed, replaced, or the `AuxWorld` is disposed.

### Query Ready

Aux ECS supports single-component and tuple-style queries which can be used in
foreach expressions.

## Known Issues

-   Accessing an `AuxRef` from a disposed `AuxWorld` will not be checked
    correctly and may lead to crashes. Be very careful when storing `AuxRef`
    instances outside of `AuxWorld` components.
-   Querying tuples results in random accesses to the underlying components, as
    Aux ECS sacrifices cache-coherency for memory stability.

## Possible Future Roadmap Items

-   Expanded API
-   Support for querying VPtr interfaces
