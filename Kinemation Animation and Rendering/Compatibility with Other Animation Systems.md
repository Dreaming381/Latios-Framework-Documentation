# Compatibility with Other Animation Systems

Kinemation bakes skinned mesh renderers in a way that is incompatible with
Unity.Deformations. For this reason, there may be some incompatibility concerns
with other animation systems. This guide will outline steps that can be taken
which may improve the compatibility of other animation solutions.

It should be noted that most other ECS animation solutions come from the Unity
Asset Store. Due to intellectual property regulations, it is not feasible to
officially develop, test, and validate compatibility layers with these other
solutions.

## Disabling Skeletal Bindings on Skinned Meshes

When you add a *Skinned Mesh Settings (Kinemation)* component to a *Skinned Mesh
Renderer*, you can set the *Binding Mode* to *Do Not Generate*. This disables
baking skeletal mesh deformation support for the entity. If the entity has blend
shapes and a deformation material, those will still be baked. Otherwise, the
entity will be baked as a regular mesh.

## Disabling Skeleton Baking

You can disable skeleton baking on any authoring Game Object with an Animator
component by adding the *Skeleton Settings (Kinemation)* component and setting
*Binding Mode* to *Do Not Generate*.

## Overriding Skinned Mesh Baking

If you wish to change how meshes, materials, and renderer settings are baked for
a skinned mesh, you can do so by adding an authoring component that implements
`IOverrideMeshRenderer` or `IOverrideHierarchyRenderer`. The latter can be on an
ancestor.

## Turning Off Baking

You can completely disable baking of skeletons and skinned meshes project-wide
by adding `SkeletonBaker` and `SkinnedMeshBaker` from the
`Latios.Kinemation.Authoring` namespace to
`CustomBakingBootstrapContext.filteredBakingTypes` inside the
`LatiosBakingBootstrap` in your bootstrap file.
