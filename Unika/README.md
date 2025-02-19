# Unika

Unika is a C\# scripting solution for Unity’s ECS. It introduces a new paradigm
that represents a middle-ground between OOP and DOD. Its purpose is to relieve
common friction points that ECS projects often face.

Unika works with **scripts**, which are user-defined struct types packed into a
`DynamicBuffer`. Scripts can implement special user-defined interfaces which can
be invoked polymorphically via source generators. With script referencing
capabilities, type and interface queries, casting, and serialization; Unika
provides a new frontier of expression for gameplay and extras.

Check out the [Getting Started](Getting%20Started.md) page!

## Features

### Jobs and Burst Ready Polymorphism

Scripts can operate within jobs and with Burst enabled, and can make full use of
their polymorphic utilities within these contexts. While virtual calls are still
not particularly performant, nearly all other benefits of DOTS are preserved,
making Unika one of the fastest scripting solutions available.

### Tech Artist and Programmer Friendly

Scripts share a lot of similarities with `MonoBehaviours`. This familiarity can
help reduce the onboarding costs of tech artists. However, scripts are more
limited in data access compared to `MonoBehaviours`, which allows the
programmers to expose only what they want to be tinkered with.

However, even programmers may find good uses for Unika for expressing gameplay.
For problems that do not require maximum performance, Unika can allow a
programmer to fall back on known OOP-style approaches and save precious
development time. Unika may even outperform an ECS-based solution on some types
of problems.

### Subscene Ready

Scripts support full baking and serialization into subscenes, including entity
references, blob assets, `UnityObjectRef<>`, and all the rest of the
ECS-specialized serializations. In addition, scripts can serialize references to
other scripts, even by interface or completely type-punned. ECS components and
dynamic buffers can also reference scripts in the same way.

*Note: Entity remapping on World change or instantiation requires a little user
intervention. However, the APIs for handling this only care about the ECS
representation and do not require the user to implement any custom logic for
specific script types.*

### Assemblies and Mod Ready

Scripts can live in separate assemblies, including assemblies that are loaded at
runtime as long as no jobs are running at that time. Unlike other ECS
polymorphism solutions, Unika does not combine all scripts implementing a
specific interface into a switch block. Instead, it uses function pointers like
real virtual calls.

### A Friend of the Caches

Despite introducing OOP concepts, the underlying storage and backend of Unika is
designed following strong data-oriented design principles. All scripts on a
single entity are packed together in memory, allowing them to interact with each
other cache-efficiently.

## Known Issues

-   Entity remapping is not automatically performed on the Scripts buffer.
    Instead, entity references within scripts need to be serialized to a
    separate buffer that Unity understands prior to remapping, and then
    deserialized after remapping. The triggering of these serializations and
    deserializations must be handled explicitly via provided systems or custom
    logic.
-   It is not possible to see specific script values in the inspector or
    debugger.
-   Polymorphic interfaces do not support `ref struct` types as parameters or
    return values. This is because `ref structs` cannot be generic arguments,
    and unsafe code compilation is not guaranteed for source generators.

## Near-Term Roadmap

-   Script Authoring Interface Serialization
-   Blob Scripts
-   Message Buffers
-   ECS Native Remap
-   Debugger Support
