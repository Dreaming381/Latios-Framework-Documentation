# Upgrade Guide [0.6.6] [0.7.0]

Please back up your project in a separate directory before upgrading! You will
likely need to reference the pre-upgraded project during the upgrade process.

## Core

### New Bootstrap and Installers

You will likely need to adjust your bootstrap to account for Unity’s switch from
`System.Type` to `SystemTypeHandle`.

### Collection and Managed Struct Components

`ICollectionComponent` and `IManagedStructComponent` have been redesigned to use
source generators. You now must declare their structs as `partial`.
Additionally, instead of an `AssociatedComponentType`, there is a
source-generated `ExistComponent` defined as a nested `IComponentData` struct
inside of your `ICollectionComponent` or `IManagedStructComponent`. You can
reference this type in your own code, including bakers. This switch to source
generators offers substantial performance and stability improvements.

### Default Systems

Instead of Latios-prefixed derived classes of the root system groups, the Latios
Framework now uses `IRateManager` on Unity’s root system groups to achieve the
same behavior. If you relied on the derived types or the ability to add a custom
`IRateManager`, you will need to rework your design.

### Transforms

The old Transforms V1 code has been completely removed. You must now use the
QVVS Transforms module.

## Psyshock Physics

### Collision Layer Source Indices

Source indices are now included inside the `CollisionLayer`, and all API to
generate them separately has been removed. Other properties were renamed as a
result.

### Baking Bootstrap

`InstallLegacyColliderBakers` was renamed to `InstallUnityColliderBakers` and
Latios Collider now shows up as *Custom Collider* in the Editor, as the built-in
Unity colliders will be the intended collider authoring workflow for the
foreseeable future.

### Renames

There were many small renames with properties, fields, and methods regarding
scaling, body indices, and casing. Check the Psyshock changelog for more
details.

## Kinemation

### Optimized Skeletons

The Optimized Skeleton runtime API has been completely reworked to make use of
`IAspect`. All code using the old API will need to switch to the new API. The
new API has the capability to modify individual bones in local, root, and world
space while keeping the hierarchy in sync.

### Revamped Archetypes

Some archetype details used for the binding processes have been reworked to
accommodate other types of deform meshes besides skeletal skinning. Any code
that didn’t just let baking handle everything will likely need to adjust for
these changes.

### Culling

The culling pipeline was reworked to improve job scheduling and sync point total
latency using a round-robin scheduling strategy. Custom culling code may need to
be reworked.

### AclUnity

AclUnity has been upgraded to the QVVS model, and consequently the APIs have
been updated accordingly. Clips now store a metadata header internally that is
parsed by the C\# methods to make the correct native calls.

## New Things to Try!

### QVVS Transforms

A new transform system might seem scary. But the new transform system is fast,
lean, and expressive. It features smaller archetypes, more features such as
stretch (non-uniform scale with less problems), and enables a whole bunch of
cool things across the other modules. Use `WorldTransform` for reading
transforms, and use `TransformAspect` when writing in local or world space. Give
it a fair shot before you judge it!

### Psyshock Physics

Thanks to the QVVS Transforms, colliders now react to transform scale and
stretch automatically. You can still manually scale and stretch colliders, but
you shouldn’t need to do so do to animated transforms.

### Kinemation

Large skeletons, blend shapes, meshes you can animate in Burst jobs, a brand new
optimized skeleton API with a synchronized hierarchy in local, root, and world
space all at the same time, render enable bits, motion history rendering
control, inertial blending, Kinemation has so many new features to try!

And that’s not considering the performance improvements across the rendering
stack thanks to QVVS Transforms!
