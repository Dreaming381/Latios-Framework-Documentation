# Upgrade Guide [0.12.7] [0.13.0]

If you are not using version control, please back up your project in a separate
directory before upgrading! You may need to reference the pre-upgraded project
during the upgrade process.

This document only highlights the common things to be aware of when upgrading.
See the changelogs of each module for a complete list of changes.

## Transforms

### Motion History Changes

Motion History has been broken up into two separate phases. The first phase is
the initialization phase for new entities, and updates in `PostSyncPointGroup`
like the old updates did. The second phase is the update phase, which now
updates at the beginning of `SimulationSystemGroup`. The change ensures that
valid motion history exists at the end of `InitializationSystemGroup`, but that
motion history is not updated right before rendering in n-1 mode. You may need
to adjust system scheduling orders accordingly.

### Extra Update

If you used QVVS Transforms and used `PreTransformSuperSystem` or
`PostTransformSuperSystem`, be aware that these super systems now update twice
each frame by default.

## Psyshock

### FindObjects

If you used `ref readonly` in a `FindObjects` `foreach` expression, you will
encounter errors when upgrading. Just delete the `ref readonly` and you should
be good to go.

## Myri

### Runtime Component Changes

The runtime components `AudioSourceLooped` and `AudioSourceOneShot` are no
longer in Myri. They have instead been replaced with `AudioSourceClip`,
`AudioSourceVolume`, and `AudioSourceDistanceFalloff`. Whether or not a clip
loops is now a property of `AudioSourceClip`. Otherwise, you’ll find all the
other fields from the old components still present. They’ve just been moved to
the new modular structure.

### Simplified Audio Listener Profile

There’s a new Audio Listener Profile, and that means the
`ListenerProfileBuilder` API has also changed. There’s no longer a concept of
“passthrough” for each channel. Instead, the signal in a given channel always
flows through all effects. You can recreate the effect of passthrough by scaling
the gain and quality values down.

### AudioClipOverrideBase Replaced with Interface

Myri’s baker will now search for instances of `IAudioClipOverride` to receive a
`SmartBlobberHandle<AudioClipBlob>` and will automatically use it. This replaces
both `AudioClipOverrideBase` and `AudioClipOverrideRequest`. If you need to
build the clip in your own baker, then simply return `default`, and then in your
own baker, you can add `AudioSourceClip` yourself.

### Voice Combining

If you spawned multiple audio sources at the same time using the same clip, you
will find the sources may be quieter after upgrading. This is because a bug was
fixed that caused voice combining to be louder than it should have been. To
restore the original volume, multiply the volume of each source by the total
number of nearby sources using the same clip.

### A New World

Don’t be alarmed if you start seeing a new ECS World show up when using Myri.
Myri is creating this for internal purposes and you should ignore it.

## Kinemation

### Motion History Update

Kinemation’s system ordering has been updated to reflect the changes to motion
history. Most projects will not be affected negatively regarding Kinemation, but
if you did something special with update ordering and Kinemation components, be
wary of these changes.

## New Things to Try!

### Core

You can now construct a `BlackboardEntity` type out of a `SystemHandle`, giving
you a much richer API for working with system entities.

### Psyshock Physics

Psyshock brings the new `CollisionWorld` type, which stores archetype info along
with a `CollisionLayer` to allow for combo ECS-spatial queries. But if that
isn’t what you are after, then maybe the new `SubstepRateManager` or
experimental `TerrainCollider` might interest you.

### Myri

Myri received a major feature update including a brand new mixing workflow,
modular audio source components, and even some rudimentary pitch shifting via
the sample rate multiplier.

### LifeFX

LifeFX now has Tracked Transforms, which provides a way for particles in VFX
Graph to track an entity’s transform and detect when that entity is destroyed.
This feature includes both ECS components to facilitate tracking, and VFX
Subgraphs included as package samples for working with the QVVS transforms.
