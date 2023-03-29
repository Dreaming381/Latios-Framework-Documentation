# Recipe 1: Request Any Reactive Pattern

Let’s say you have a zero-sized `IComponentData` called `SuperCoolTag`. This
component is very important, and so many bakers want to ensure it gets added.
So, there's this one `GameObject` that has two authoring components. Each of
those components has an associated baker that tries to add `SuperCoolTag`. Guess
what Unity does? It freaks out!

## Idea

The solution to this is to make every baker define its own variant of
`SuperCoolTag`, such as `BakerASuperCoolTag` and `BakerBSuperCoolTag`, which are
both decorated with `[BakingType]`. Next, a baking system queries for **any** of
`BakerASuperCoolTag` or `BakerBSuperCoolTag` and adds the `SuperCoolTag`
component to all entities matching that query. But, we also need to be able to
undo this operation. For that, we specify a second query with **all** of
`SuperCoolTag` and **none** of `BakerASuperCoolTag` and `BakerBSuperCoolTag`.
For such a query, we remove `SuperCoolTag`.

Don't forget to add `Disabled` and `Prefab` into your `EntityQueryOptions` for
both queries!

## Extending

It is a simple solution, and it handles the "phantom components" correctly. But
every time there's a new baker with a new `SuperCoolTag` variant, the baking
system needs to be updated. That isn't always possible, because the bakers might
be in different assemblies or the baking system may need to be shipped as part
of a package.

Fortunately, this solution can be somewhat generalized. The baking system simply
needs to define an interface like this:

```
[BakingType]
public interface ISuperCoolTagRequest : IComponentData {}
```

Every baker then defines an ICD that implements this interface.

Now, in `OnCreate()` of the baking system, search through all component types in
`TypeManager` and find the ones that implement this interface and store them in
an array. Then use that array to build the `Any` query, and the `None` query.
After that, you should never have to touch the baking system again.

```csharp
public struct SuperCoolTag : IComponentData { }

[BakingType]
public interface ISuperCoolTagRequest : IComponentData { }

class BakerA : Baker<AuthoringA>
{
    struct BakerASuperCoolTag : ISuperCoolTagRequest { }
    
    public override void Bake(AuthoringA authoring)
    {
        AddComponent<BakerASuperCoolTag>();
    }
}

class BakerB : Baker<AuthoringB>
{
    struct BakerBSuperCoolTag : ISuperCoolTagRequest { }

    public override void Bake(AuthoringB authoring)
    {
        AddComponent<BakerBSuperCoolTag>();
    }
}

[WorldSystemFilter(WorldSystemFilterFlags.BakingSystem)]
[RequireMatchingQueriesForUpdate]
[BurstCompile]
partial struct SuperCoolBakingSystem : ISystem
{
    EntityQuery m_addQuery;
    EntityQuery m_removeQuery;

    // Cache this across all worlds. Will get rebuilt on domain reload.
    static List<ComponentType> s_requestTypes;

    public void OnCreate(ref SystemState state)
    {
        if (s_requestTypes == null)
        {
            s_requestTypes = new List<ComponentType>();
            var interfaceType = typeof(ISuperCoolTagRequest);

            foreach (var type in TypeManager.AllTypes)
            {
                if (!type.BakingOnlyType)
                    continue;
                if (interfaceType.IsAssignableFrom(type.Type))
                    s_requestTypes.Add(ComponentType.ReadOnly(type.TypeIndex));
            }
        }

        var typeList = s_requestTypes.ToNativeList(Allocator.Temp);

        m_addQuery = new EntityQueryBuilder(Allocator.Temp).WithAny(ref typeList).WithNone<SuperCoolTag>().WithOptions(EntityQueryOptions.IncludePrefab | EntityQueryOptions.IncludeDisabledEntities).Build(ref state);
        m_removeQuery = new EntityQueryBuilder(Allocator.Temp).WithAll<SuperCoolTag>().WithNone(ref typeList) .WithOptions(EntityQueryOptions.IncludePrefab | EntityQueryOptions.IncludeDisabledEntities).Build(ref state);
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        state.CompleteDependency();

        state.EntityManager.AddComponent<SuperCoolTag>(m_addQuery);
        state.EntityManager.RemoveComponent<SuperCoolTag>(m_removeQuery);
    }

    [BurstCompile]
    public void OnDestroy(ref SystemState state)
    {
    }
}
```

## Data Writes

Now what if `SuperCoolTag` was no longer zero-sized? What if it became
`SuperCoolData`?

The pattern still works, but you need to compute the values of `SuperCoolData`
somehow. You can do that with an `IJobEntity` after adding and removing the
component from the two queries. If you recompute the values of all instances
every update, you will never have artifacts. However, checking both change
versions and order versions and operating on all changes should give you a
little more editor performance. **Do not try to track archetype changes!** You
will mess up. It is better to be conservative and recompute the values unless
you are absolutely sure nothing changed since the last incremental update.

In practice, I often find that `SuperCoolData` just needs to be
default-initialized, because it represents runtime state. That's just as easy as
the zero-sized case.

## Buffer merging

Finally, there's one more use case where this technique can be applied. And that
is merging dynamic buffers. You still have to clear and remerge all the buffers
every update (or using change and version filters). But that can be
parallelized.

```csharp
[InternalBufferCapacity(0)]
public struct BufferElement : IBufferElementData
{
    public Entity entity;
}

class BufferBakerA : Baker<ABufferAuthoring>
{
    [BakingType]
    public struct RequestInBuffer : IComponentData
    {
        public Entity entity;
    }

    public override void Bake(ABufferAuthoring authoring)
    {
        AddComponent(new RequestInBuffer { entity = GetEntity(authoring.someReferencedGameObject) });
    }
}

class BufferBakerB : Baker<BBufferAuthoring>
{
    [BakingType]
    public struct RequestInBuffer : IComponentData
    {
        public Entity entity;
    }

    public override void Bake(BBufferAuthoring authoring)
    {
        AddComponent(new RequestInBuffer { entity = GetEntity(authoring.someReferencedGameObject) });
    }
}

[WorldSystemFilter(WorldSystemFilterFlags.BakingSystem)]
[RequireMatchingQueriesForUpdate]
[BurstCompile]
partial struct BufferBakingSystem : ISystem
{
    EntityQuery m_addQuery;
    EntityQuery m_removeQuery;
    EntityQuery m_anyQuery;

    public void OnCreate(ref SystemState state)
    {
        m_addQuery = new EntityQueryBuilder(Allocator.Temp).WithAny<BufferBakerA.RequestInBuffer, BufferBakerB.RequestInBuffer>().WithNone<BufferElement>().WithOptions(EntityQueryOptions.IncludePrefab | EntityQueryOptions.IncludeDisabledEntities).Build(ref state);
        m_removeQuery = new EntityQueryBuilder(Allocator.Temp).WithAll<BufferElement>().WithNone<BufferBakerA.RequestInBuffer, BufferBakerB.RequestInBuffer>().WithOptions(EntityQueryOptions.IncludePrefab | EntityQueryOptions.IncludeDisabledEntities).Build(ref state);
        m_anyQuery = new EntityQueryBuilder(Allocator.Temp).WithAny<BufferBakerA.RequestInBuffer, BufferBakerB.RequestInBuffer>().WithOptions(EntityQueryOptions.IncludePrefab | EntityQueryOptions.IncludeDisabledEntities).Build(ref state);
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        state.CompleteDependency();

        state.EntityManager.AddComponent<BufferElement>(m_addQuery);
        state.EntityManager.RemoveComponent<BufferElement>(m_removeQuery);

        state.Dependency = new Job
        {
            bufferHandle = SystemAPI.GetBufferTypeHandle<BufferElement>(false),
            aHandle = SystemAPI.GetComponentTypeHandle<BufferBakerA.RequestInBuffer>(true),
            bHandle = SystemAPI.GetComponentTypeHandle<BufferBakerB.RequestInBuffer>(true),
            lastSystemVersion = state.LastSystemVersion
        }.ScheduleParallel(m_anyQuery, state.Dependency);
    }

    [BurstCompile]
    public void OnDestroy(ref SystemState state)
    {
    }

    [BurstCompile]
    struct Job : IJobChunk
    {
        public BufferTypeHandle<BufferElement> bufferHandle;
        [ReadOnly] public ComponentTypeHandle<BufferBakerA.RequestInBuffer> aHandle;
        [ReadOnly] public ComponentTypeHandle<BufferBakerB.RequestInBuffer> bHandle;
        public uint lastSystemVersion;

        public void Execute(in ArchetypeChunk chunk, int unfilteredChunkIndex, bool useEnabledMask, in v128 chunkEnabledMask)
        {
            bool hasA = chunk.Has(ref aHandle);
            bool hasB = chunk.Has(ref bHandle);
            
            bool anythingChanged = chunk.DidOrderChange(lastSystemVersion);
            anythingChanged |= hasA && chunk.DidChange(ref aHandle, lastSystemVersion);
            anythingChanged |= hasB && chunk.DidChange(ref bHandle, lastSystemVersion);
            if (!anythingChanged)
                return;

            var buffers = chunk.GetBufferAccessor(ref bufferHandle);
            var aArray = chunk.GetNativeArray(ref aHandle);
            var bArray = chunk.GetNativeArray(ref bHandle);
            for (int i = 0; i < chunk.Count; i++)
            {
                var buffer = buffers[i].Reinterpret<Entity>();
                
                // We have to rebuild the buffer from scratch every time. Don't try to incrementalize this.
                buffer.Clear();

                if (hasA)
                    buffer.Add(aArray[i].entity);
                if (hasB)
                    buffer.Add(bArray[i].entity);
            }
        }
    }
}
```

So there you have it. You can now start merging tags and dynamic buffers safely
using a baking system!
