# VPtr

VPtr is a source-generator solution for adding polymorphic behavior to structs
implementing interfaces in a Burst-compatible way. This is useful for
power-users who want to store interface-implementing structs in raw memory (such
as a `NativeStream`) and invoke the interface methods later, or for passing to
utilities without the need of making the full code path rely on
interface-constrained generics.

## Defining an Interface

In order to create `VPtr`s, you need to define an interface. The interface
should be declared as partial, and directly inherit `IVInterface` (that is, you
need to type that in the interface declaration so that source-generators find
your interface).

The interface can declare methods, properties, and indexers. All parameters and
return values must be unmanaged types, and cannot be `ref struct`s. However,
parameters support `in`, `ref`, and `out` modifiers. And return values can be
`ref` and `readonly ref`. Generics are not supported.

## VPtr and VPtrFunction

With the interface defined, the source generator will add additional code to the
interface to define two public nested types: `VPtr` and `VPtrFunction`.

`VPtrFunction` is a wrapper around a Burst `FunctionPointer` that can be used to
dispatch any interface method. `VPtr` is a struct that holds a pointer and a
`VPtrFunction`, and provides the interface API which can invoke the interface
methods through the `VPtrFunction`.

You can create a `VPtr` either by passing a strongly-typed pointer to a struct
implementing the interface, or you can manually pass in a type-erased pointer
and the `VPtrFunction`. The latter can be powerful, but also potentially allows
you to call a `VPtrFunction` instance for the wrong type, which could result in
memory corruption or crashes.

The source generator will also define several static methods on the interface,
two of which are public. `GetVPtrFunctionFrom<T>()` allows getting the
`VPtrFunction` of a specific implementing `struct` type.
`TryGetVPtrFunctionFrom()` allows getting the `VPtrFunction` from the result of
`BurstRuntime.GetHashCode64<T>()` where `T` is the implementing `struct` type.

## Defining an Implementing Struct

Each struct that implements an interface derived from `IVInterface` also
requires source-generated code. This means, the struct must be defined as
partial, and must additionally declare that it implements `IVInterface`
alongside the deriving interface.

## Other Notes

Unlike other polymorphic solutions, `VPtr` does not use a union struct, nor does
it require all implementing types be within the same assembly. This means that
it can support loading mods dynamically.
