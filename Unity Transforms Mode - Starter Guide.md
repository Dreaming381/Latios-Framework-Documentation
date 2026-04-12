# Unity Transforms Mode – Starter Guide

There are a lot of features in the Latios Framework. And if you are using Unity
Transforms, it might be confusing knowing which features are good to use, and
which are generally geared more towards QVVS users. That’s what this guide is
for.

We’ll start with the most popular modules and features for Unity
Transforms-based projects, and then discuss other features for the more
adventurous. This order is largely based on conversations from the Latios
Framework discord community.

## Kinemation

If you are interested in the Latios Framework because you want a free skeletal
animation solution, welcome! Some people use the Latios Framework solely for
this purpose. You are in good company.

If you aren’t interested in Kinemation for skeletal animation, don’t write off
Kinemation just yet. That’s just one of Kinemation’s many features.

Really, you should think of Kinemation as an upgraded Entities Graphics. It is
just plain faster (both CPU and GPU), and offers many more features.

If you have a lot of entities, LOD Pack can make a huge impact on your polygon
count. If you have a lot of textures, mipmap streaming works out of the box. And
if you just want to create and modify a mesh at runtime, Unique Mesh makes that
process so much easier.

## Calligraphics

Calligraphics does world-space text at scale really well. If you are worried
about needing 1000 canvases or UI Toolkit panels, then Calligraphics is
solution.

The most common use case for Calligraphics is for damage numbers. But there are
other use cases where a bunch of independent world-space text labels are needed.

## LifeFX

If you’ve ever explored Unity’s Galaxy Sample, you might be familiar with its
technique of using a single VFX Graph to service many entities. LifeFX works in
a similar way, but provides a better workflow for wiring up entities to the
Visual Effects.

## Explicit System Ordering

Explicit System Ordering is popular enough that it is this high in the list.
Solo devs in particular really like just being able to see the order of their
systems right in the code.

## Myri

Myri is such an easy drop-in for audio when you need something simple. The only
catch is that the listener needs to be an entity that gets baked, which means it
cannot be the Main Camera Game Object.

For users with more-advanced audio needs, Myri is still a great tool for
prototyping at the early stages of development, before switching over to FMOD
Studio or Wwise.

## DynamicHashMap

This hashmap-in-a-`DynamicBuffer` solution was just a side-experiment. But
people really like it. It’s main claim to fame is that the value type in a
key-value pair supports `Entity` and `BlobAssetReference` serialization.

## Unika

Unika comes in handy for problems where you really, really miss virtual
interface calls. It is good for things like state machines and behavior trees.
But you can also use it as a C\# scripting solution for mods. And for small
gameplay logic tasks, it is faster to work with Unika than to make a full ECS
component and system once you have it set up.

## Calci

Calci is the home of Rng, which is a deterministic random number generator that
is simpler to set up than other solutions.

## And More…

There are plenty more features compatible with Unity Transforms, but these are
the ones that come up the most. If you use the framework, be sure to share which
features you use, as doing so helps ensure those features and better maintained.

## TransformQvvs Side-Note

When using Unity Transforms, you will still come across `TransformQvvs` as part
of the API. This is just a struct, not unlike `RigidTransform` from the Unity
Mathematics package. Do whatever you want with the individual fields.
