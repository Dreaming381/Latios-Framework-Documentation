# QVVS Transforms Baking

QVVS Transforms operate a little differently from Unity’s Transform System, and
consequently have slight differences in how baking operates. QVVS Transforms use
two separate baking systems, one that processes *tags* and the other that
processes *flags*.

## Tags

There are three optional transform components that can be added to an entity:
`PreviousTransform`, `TwoAgoTransform`, and `CopyParentWorldTransformTag`. All
three components can be added using the [Request Any Reactive
Pattern](../Level%20Up%20Your%20ECS/Baking%20Systems%20Recipes/Part%202%20-%20Request%20Any%20Reactive%20Pattern.md)
inside a baker. Adding a `CopyParentWorldTransformTag` inside a baker may be
done as follows:

```csharp
class MyCopyParentTransformAuthoring : UnityEngine.MonoBehaviour { }

class MyBaker : Baker<MyCopyParentTransformAuthoring>
{
    [BakingType]
    struct MyCopyParentRequestTag : IRequestCopyParentTransform { }

    public override void Bake(MyCopyParentTransformAuthoring authoring)
    {
        AddComponent<MyCopyParentRequestTag>();
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
removes the following four components: `WorldTransform`, `LocalTransform`,
`ParentToWorldTransform`, and `Parent`.

### None

The `TransformBakingSystem` will remove all four of the considered components.

### Renderable

A `Renderable` entity will always receive a `WorldTransform` component. If the
entity does not have a parent that has the `Dynamic` flag nor has the `Dynamic`
flag itself, then `WorldTransform` will be the only component of the four to
exist once the baking system completes. Otherwise, it is treated as if it were
`Dynamic`. This is a good flag for bakers that know the transform will need to
be read but may or may not need to move.

### Dynamic

A `Dynamic` entity will always receive a `WorldTransform` component. If it has a
parent, it will also receive a `Parent` component. However, only if it has a
parent but does not have the `CopyParentWorldTransformTag` (which is set up by
the previous baking system) will it receive the `LocalTransform` and
`ParentToWorldTransform` components. Otherwise, those latter two components will
be removed. If the entity does not have a parent, the `Parent` component will be
removed.

### WorldSpace

A `WorldSpace` entity will always receive a `WorldTransform` component, and the
other three components will always be removed. If the entity has the `Static`
component, this flag is implied.

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

## HierarchyUpdateMode

`HierarchyUpdateMode` uses a flag system similar to `TransformUsageFlags`, and
can be added to any entity in a baker, including entities obtained via
`GetEntity()`. To do so, simply call the `IBaker.AddHierarchyModeFlags()`
extension method and pass in the desired world-space lock flags. All requests
targeting a specific entity will be combined with bitwise-OR logic.

## Systems

You can add systems to the `UserPreTransformsBakingSystemGroup` to have systems
which add tag components or set up transforms manually but should interact with
the rest of the hierarchy.
