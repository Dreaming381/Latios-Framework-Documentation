# Blackboard Entities

Blackboard entities are entities automatically created by the world for the
purposes of attaching arbitrary components to centralized entities or
“blackboards”.

## SceneBlackboardTag and WorldBlackboardTag

Every world maintains two blackboard entities.

-   The `sceneBlackboardEntity` has the `SceneBlackboardTag` component attached.
-   The `worldBlackboardEntity` has the `WorldBlackboardTag` component attached.

While it is possible to add or remove these components, doing so is not advised.
Like-wise instantiating and deleting these entities could also cause serious
problems. Please allow these entities and their associated tag components to be
handled by the framework.

You may however find it advantageous to use these tags in queries like in the
following example:

```csharp
[BurstCompile]
public void OnUpdate(ref SystemState state)
{
    var dcb = latiosWorld.syncPoint.CreateDestroyCommandBuffer();

    var matchingCard = latiosWorld.sceneBlackboardEntity.GetComponentData<Card>();

    foreach ((var card, var entity) in SystemAPI.Query<Card>().WithEntityAccess().WithNone<SceneBlackboardTag>())
    {
        if (card.Equals(matchingCard))
        {
            dcb.Add(entity);
        }
    }
}
```

## BlackboardEntity Type

`BlackboardEntity` is a struct type which holds an `Entity` and an unmanaged
reference to the world it belongs to. The struct provides convenient methods for
operating directly on the entity as well as an implicit cast to the `Entity`
type. The underlying `EntityManager` access is hidden, so you can pass a
`BlackboardEntity` to a method without access to an `EntityManager` and expect
that the method will only ever be able to modify the `BlackboardEntity` and no
other entity.

You can access `sceneBlackboardEntity` and `worldBlackboardEntity` properties in
the following ways:

```csharp
public struct CompA : IComponentData
{
    public int count;
}

public partial class ManagedSystem : SubSystem
{
    protected override void OnUpdate()
    {
        var a = worldBlackboardEntity.GetComponentData<CompA>();
        a.count++;
        worldBlackboardEntity.SetComponentData(a);
    }
}

[BurstCompile]
public partial struct UnmanagedSystem : ISystem
{
    LatiosWorldUnmanaged latiosWorld;

    [BurstCompile]
    public void OnCreate(ref SystemState state) 
    {
        latiosWorld = state.GetLatiosWorldUnmanaged();
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var a = latiosWorld.worldBlackboardEntity.GetComponentData<CompA>();
        a.count++;
        latiosWorld.worldBlackboardEntity.SetComponentData(a);
    }
}

public partial class BlackboardSuperSystem : SuperSystem
{
    protected override void CreateSystems()
    {
        GetOrCreateAndAddSystem<ManagedSystem>();
        GetOrCreateAndAddUnmanagedSystem<UnmanagedSystem>();
    }

    public override bool ShouldUpdateSystem()
    {
        return worldBlackboardEntity.GetComponentData<CompA>().count < 10;
    }
}

public class MonoBridge : UnityEngine.MonoBehaviour
{
    private void Update()
    {
        var latiosWorld = World.DefaultGameObjectInjectionWorld as LatiosWorld;
        latiosWorld.worldBlackboardEntity.SetComponentData(new CompA { count = 0 });
    }
}
```

## Blackboard Entity Lifetimes

The `worldBlackboardEntity`’s lifetime is tied to the `LatiosWorld` that created
it.

In contrast, the `sceneBlackboardEntity`’s lifetime is tied to the active scene
in the [Scene Management System](Scene%20Management.md).

*Note: You may observe multiple entities named “SceneBlackboardEntity” in the
Entity Debugger. Only one of these entities should have the*
`SceneBlackboardTag` *attached. The rest are likely entities with System State
types waiting to be cleaned up.*

## Authoring Blackboard Entities

You can author blackboard entities using the `BlackboardEntityData` component or
its authoring equivalent:

![](media/66f6dfe6742192bcd39b47de654e593f.png)

In the `LatiosWorldSyncGroup`, the `MergeBlackboardsSystem` will iterate through
each entity with the `BlackboardEntityData` component and copy the entity’s
other components to one of the blackboard entities based on the following
settings:

-   Blackboard Scope – The target entity to copy the components to
    -   Scene – Copy the components to the `sceneBlackboardEntity`
    -   World – Copy the components to the `worldBlackboardEntity`
-   Merge Method – The logic to apply for each component shared by both the
    source entity and target blackboard entity
    -   Overwrite – The source entity’s component value will replace the
        blackboard entity’s component value
    -   Keep Existing – The blackboard entity’s component value will be left
        unchanged
    -   Error On Conflict – If a component is shared by both entities, an
        exception will be thrown

Some additional rules exist regardless of the settings:

-   Any components attached exclusively to the blackboard entity will remain
    attached
-   Any components attached exclusively to the source entity will be added to
    the blackboard entity with matching values with the exception of
    `ICollectionComponent` and `IManagedComponent` which will always be ignored
-   The source entity will always be destroyed
-   Disabled and Prefab entities with the `BlackboardEntityData` component will
    be ignored, allowing you to enable or instantiate these entities at runtime
    to alter the blackboard entities
