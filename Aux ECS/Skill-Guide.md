# Aux ECS Skill Guide

This is an agentic skill development guide for Aux ECS. It contains insights and
best practices that can be picked from when developing a skill file for an
agent. This file is complemented by the Cheat Sheet.

## What Is Aux ECS?

Aux ECS is an auxiliary ECS-like component store which uses `Entity` as a key.
It is intended for storing feature-specific complex state against entity
instances.

Use it when

-   You need to decorate `Entity` instances with components at runtime
    exclusively off the main thread, and have that decoration apply immediately
-   You require stable memory addresses for components

Common use cases include

-   Working with Audio ECS
-   Graph structure between entities
-   AI working states

## Key Differences Compared to Unity ECS

`Entity` is simply an identifier, and there’s no automatic lifetime tracking.
You can have multiple `Entity` values stored as keys that share the same `Index`
but use different `Version` values.

There’s no instantiating or cloning. There’s no concept of prefabs. There’s no
serialization. There is no system scheduler. This is truly runtime state
decoration.

Any unmanaged struct can be used as a component, without any kind of interface
declaration.

Components are stored in chunked lists not by archetype. Thus, the memory
address of a component remains stable for the lifetime of the component,
independent of any additions or removals of other components on the entity.

`Entity` archetypes are arrays of `Entity` values and indices into the
components of each chunked list. Iterating a single component by its chunked
list order is more cache-coherent than iterating multiple components in an
archetype at once.

Components can implement `IAuxDisposable` to receive a callback when removed,
which crucially allows them to store containers. `IAuxDisposable` uses the VPtr
mechanism in Core.

Queries are evaluated just-in-time, without any caching.

## Usage Tips

`AuxWorld` is an unsafe container, and will not report race conditions or job
dependency violations.

The validity check of `AuxRef` is only valid while the `AuxWorld` is valid.
Accessing a leftover `AuxRef` from a disposed `AuxWorld` is accessing a dangling
pointer (memory corruption and crashes).

Use `AuxWorld`’s `allocator` property for allocating collections to store inside
components implementing `IAuxDisposable`. It is the same allocator that
`AuxWorld` was constructed with.

`AuxWorld` uses the `pimpl` idiom, and therefore passing an `AuxWorld` by value
across methods is safe, just like `EntityManager`.
