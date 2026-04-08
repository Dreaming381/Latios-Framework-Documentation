# QVVS Transforms Baking

QVVS Transforms operate a little differently from Unity’s Transform System, and
consequently have slight differences in how baking operates. QVVS Transforms use
two separate baking systems, one that processes *tags* and the other that
processes *flags*.

## Tags

There are two optional transform components that can be added to an entity:
`PreviousTransform`, and `TwoAgoTransform`. Both components can be added using
the [Request Any Reactive
Pattern](../Level%20Up%20Your%20ECS/Baking%20Systems%20Recipes/Part%202%20-%20Request%20Any%20Reactive%20Pattern.md)
inside a baker. Adding a `PreviousTransform` inside a baker may be done as
follows:

```csharp
class MyPreviousTransformAuthoring : UnityEngine.MonoBehaviour { }

class MyBaker : Baker<MyPreviousTransformAuthoring>
{
    [BakingType]
    struct MyPreviousTransformRequestTag : IRequestPreviousTransform { }

    public override void Bake(MyPreviousTransformAuthoring authoring)
    {
        AddComponent<MyPreviousTransformRequestTag>();
    }
}
```

For `PreviousTransform`, your component needs to implement the
`IRequestPreviousTransform` interface. And consequently for `TwoAgoTransform`,
your component must implement the `IRequestTwoAgoTransform` interface. These
motion history components will be initialized at runtime in the
`MotionHistoryUpdateSuperSystem` which runs early in the
`SimulationSystemGroup`.

## Flags

QVVS Transform baking accounts for `TransformUsageFlags`. However, the meaning
of these flags is slightly different with QVVS Transforms. It only adds or
removes the following three components: `WorldTransform`, `RootReference`, and
`EntityInHierarchy`.

### None

The `TransformBakingSystem` will remove all three of the considered components.

### Renderable

A `Renderable` entity will always receive a `WorldTransform` component. If the
entity does not have a parent that has the `Dynamic` flag nor has the `Dynamic`
flag itself, then it is treated as if it was assigned the `WorldSpace` flag.
Otherwise, it is treated as if it was assigned the `Dynamic` flag.

### Dynamic

A `Dynamic` entity will always receive a `WorldTransform` component. If it has a
parent, it will also receive a `RootReference` component. If the entity does not
have a parent, the `RootReference` component will be removed. If it does not
have a parent but has a child, then `EntityInHierarchy` will be added.
Otherwise, `EntityInHierarchy` will be removed.

### WorldSpace

A `WorldSpace` entity will always receive a `WorldTransform` component, and the
`RootReference` component will always be removed. The entity might receive an
`EntityInHierarchy` if it has a child. If the entity has the `Static` component,
this flag is implied. A `WorldSpace` entity is never counted as a child of
another entity.

### NonUniformScale

This flag has no effect on QVVS baking. However, specifying this flag will
inform the Kinemation renderer to add a `PostProcessMatrix` component. Stretch
(a form of non-uniform scale) is built into `WorldTransform`.

### ManualOverride

The baking system will skip processing of all entities with this flag. This
comes with one side-effect, in that it will also fail to remove non-requested
components, which may result in incorrect results during incremental baking.
Such results will be resolved upon closing the subscene. A workaround for this
issue is planned for a future release.

## Inheritance Flags

`InheritanceFlags` baking uses a flag system similar to `TransformUsageFlags`,
and can be added to any entity in a baker, including entities obtained via
`GetEntity()`. To do so, simply call the `IBaker.AddInheritanceFlags()`
extension method and pass in the desired world-space lock flags. All requests
targeting a specific entity will be combined with bitwise-OR logic.

## Partial Overrides

You can override the local transform or parent in a baker without using
`ManualOverride`. To do so, add the `BakedLocalTransformOverride` or
`BakedParentOverride` components respectively, and initialize their values to
what you want them to be.

## Order Preservation

Unless you override the parent, the order of siblings of a Game Object are
preserved when baked into entities. Entities that do not come from Game Objects
will follow all those that do in the sibling sequence.

## Systems

You can add systems to the `UserPreTransformsBakingSystemGroup` to have systems
which add tag components or set up transforms manually but should interact with
the rest of the hierarchy.
