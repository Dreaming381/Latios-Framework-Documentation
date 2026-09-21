# Aux ECS Cheat Sheet

-   `AuxWorld` – Storage and primary API type
    -   `allocator` – The allocator to use for `IAuxDisposable`
    -   `AddComponent<T>()` and `RemoveComponent<T>()`
    -   `RemoveAllComponents()` – For when entity is no longer needed
    -   `TryGetComponent<T>()` – Always read-write access
    -   `AllOf<T>()` – Query all components of type in component chunked list
        order
    -   `AllWith<T1, T2…>()` – Query matching components in archetype order
-   `AuxRef<T>` – Stable reference to a component
    -   `aux` – Returns the component by `ref`
-   `AuxComponentEnumerator<T>` – Return value of `AllOf<T>()`
    -   `count` – Number of valid components
-   `AuxQueryEnumerator<T1, T2…>` – Return value of `AllWith<T1, T2…>()`
    -   `Count()` – Number of entities matching the query, counted dynamically
-   `IAuxDisposable` – `IVInterface` invoked when component is about to be
    removed (including `AuxWorld` disposal) or replaced
