# Dynamic Hashmap

If you ever wanted a per-entity hashmap with the automatic memory management of
`DynamicBuffer`, `DynamicHashMap` has you covered. `DynamicHashMap` is a hashmap
embedded inside a `DynamicBuffer`. While there are several such implementations
available on the internet, the Latios Framework flavor supports proper subscene
serialization and entity remapping for the Values of Key-Value pairs.

## Usage

To create a `DynamicHashMap`, define an `IBufferElementData` which contains a
single field of the nested type `Pair` for the specific `DynamicHashMap` pairing
you desire.

```csharp
[InternalBufferCapacity(0)]
struct CellToEntity : IBufferElementData
{
    public DynamicHashMap<int2, Entity>.Pair element;
}
```

Now that you have this, you can add and remove a `DynamicBuffer` of this type to
various entities.

To work with the `DynamicBuffer` as a `DynamicHashMap`, you must construct a
`DynamicHashMap` object in each context you wish to use it. `DynamicHashMap` is
simply a lightweight wrapper struct around the `DynamicBuffer`, referencing the
`DynamicBuffer`. You must reinterpret the `DynamicBuffer` to the underlying
`Pair` type in order to construct the `DynamicHashMap`.

```csharp
var cellToEntityMap = new DynamicHashMap<int2, Entity>(cellToEntityBuffer.Reinterpret<DynamicHashMap<int2, Entity>.Pair>());
```

After this, you will have various methods at your disposal to work with the
hashmap.

You can use `foreach` on a `DynamicHashMap` to iterate all pairs.

## PairRef Hazards

The methods `GetOrAdd()` and `TryGet()` provide an instance of type `PairRef`.
`PairRef` stores a reference to the key-value pair such that the value may
optionally be modified. In various situations, this can remove redundant key
searches. However, `PairRef` can become invalid when new elements are added, as
the following example demonstrates:

```csharp
// Adding and retrieving a reference
var entity_0_0 = cellToEntityMap.GetOrAdd(new int2(0, 0));

// Safe assignment
entity_0_0.valueRW = entityA;

// Adding a new pair if key not already present
cellToEntityMap.TryAdd(new int2(1, 1), entityB);

// No longer safe, might throw error
entity_0_0.valueRW = entityA;
```

## Advanced Usage

The values of key-value pairs can be remappable types and serialized. This means
that `Entity`, `BlobAssetReference`, and `UnityObjectRef` are all supported as
values or as fields of a value struct type. Such `DynamicHashMaps` are safe to
bake and instantiate at runtime.

However, using any of these types as keys is NOT safe. This is because the
remapping of keys will change their hashcodes, and thus elements won’t be stored
in the correct buckets. If you choose to ignore this and store one of these
types as a key anyways, you can recover the hashmap by calling
`ReconstructOnRemap()` to rehash all elements before using it.
