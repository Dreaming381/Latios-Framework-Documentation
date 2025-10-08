# Getting Started with Myri Audio

Myri is an audio solution for Unity ECS. It drew inspiration from the Megacity
2019 demo, but has since gone above and beyond. It currently supports simple
production use cases where only spatialization and simple mixing are required.
Advanced sound design tools are not yet present, though they are planned for a
future release. If you are a sound designer and would like to help shape Myri’s
feature, please reach out!

With that said, Myri is still a production-ready solution for many styles of
games. And what it may lack in features, it makes up for in simplicity and
performance.

## Setting up the Managed Driver

Many people encounter an issue with domain reload freezing when using Myri
out-of-the-box, forcing a restart of the editor to continue progress. If you
encounter this, you can go to the edit menu and select *Latios -\> Use Myri
Editor Managed Driver*. This setting is unique per computer per project.
Enabling this has two side-effects.

First, it requires a Unity Engine Audio Listener attached to the main camera.
This is the only time a Unity Engine audio component is ever used in Myri.

Second, the managed driver will always attach the Mono runtime and garbage
collector to the DSP mixing thread. This doesn’t happen with the default Burst
driver.

With these two alterations, the managed driver may exhibit different behavior in
the editor when switching scenes or doing any other custom loading compared to
in a build.

## Playing Your First Sound

Create a new Unity Project with the Latios Framework installed. Then add the
*Latios Bootstrap* using the *Standard – Injection Workflow*.

Create a subscene, and add two empty Game Objects to it with their positions set
to the world origin (0, 0, 0). On one of the Game Objects, in the *Add
Component* *Menu*, select *Latios -\> Myri -\> Audio Listener (Myri)*. Leave all
the settings at default.

Next, import an audio source with the *Load Type* set to *Decompress On Load*
and *Preload Audio Data* checked (the latter is optional on some platforms).
*Load in Background* should be unchecked. Then, on the non-listener Game Object,
add a *Latios -\> Myri -\> Audio Source (Myri)* component. Drag your clip into
the *Clip* field. Now enter play mode. You should hear your clip playing.

Now that you’ve heard your clip, the next thing to do is to start playing around
with the position of the audio source. If you move your audio source to -1 on
the x-axis, the clip should sound like it is playing primarily through your left
ear. If you move the source more than 5 units away, it will start to get
quieter.

What you also may have realized, is that the clip always starts playing right
away, and there’s no setting to change that. At runtime, if you wanted to play a
sound when a specific event happens, you would do so by instantiating an audio
prefab entity at the location you want the sound to play.

## Authoring Options

Now that you understand the absolute basics of getting audio to play, let’s go
through the available options:

Myri provides three components for authoring audio. They can all be found in
*Latios -\> Myri* in the *Add Component Menu*.

### Audio Source

The *Audio Source (Myri)* component provides all the settings available to an
audio-emitting entity. At the top are the *Clip* and *Volume* controls.

An important thing to keep in mind is that audio volume and perceptual volume
are two very different scales. Myri’s authoring presents everything in raw audio
scales. Which means to make something perceptually half as loud, you’d multiply
the volume by 0.1, and to make something a quarter as loud, you’d multiply it by
0.01 (yes, it is very extreme).

For *Looping* sources, *Voices* is the number of unique audio sources that will
play with the same clip. Each unique source will be played with a unique offset.
The offsets are evenly distributed. All sources sharing the same clip will be
assigned one of these offsets, and sources sharing the same offset will be
combined. Voices are synchronized with the application time independent of when
the source was created. You would typically use these for looping ambient
sources scattered across your game world. Too many different offsets at once can
result in cacophony. If you instead want to play a source from the beginning
when it is created, check the *Play From Beginning At Spawn* checkbox. In this
case, the *Voices* parameter does nothing.

One-shot sources will be combined if they begin playing at the same time (the
same audio frame). This is purely a performance optimization, and the resulting
mix is identical to what would happen if the sources were sampled separately.

If *Auto Destroy on Finished* is checked and *Looping* is unchecked, the entity
containing the audio source will automatically be destroyed once the clip has
finished playing. This uses the [Auto-Destroy
Expirables](../Core/Auto-Destroy%20Expirables.md) mechanism.

The *Codec* lets you choose a compression format to store the clip. Compressed
clips decrease build and runtime memory usage, but require more processing power
at runtime to play. The current options are as follows:

-   Uncompressed – No compression, fast, full quality, great for short clips
    that get instantiated in large quantities
-   ADPCM – 8:1 compression ratio, medium speed and quality

The *Audio Channel* lets you specify an audio channel asset. Audio channels are
used for categorizing sources so that they can only be heard by specific
listeners. To create an audio channel asset, go to *Assets -\> Create -\> Latios
\-\> Myri (Audio) -\> Audio Channel*. If you don’t provide an audio channel
asset, the audio source will use an internal default channel.

When *Use Falloff* is enabled, Myri enables spatialization on the audio source.
When the listener is inside the *Inner Range*, the source will be heard at max
volume. Beyond that, Myri uses inverse-square falloff, so at twice the *Inner
Range*, the audio source will be heard at quarter volume. The *Range Fade
Margin* provides a range inside the *Outer Range* where the volume will
interpolate to zero.

If *Use Cone* is enabled, then the audio source receives an additional
directional attenuation factor. Listeners within the *Inner Angle* from the
source’s forward direction will not receive any attenuation. Listeners at the
*Outer Angle* will be attenuated by the *Outer Angle Volume*. *Use Cone* has no
effect if *Use Falloff* is disabled.

### Audio Listener

The *Audio Listener* component provides all the settings for an entity that is
able to receive and spatially process sounds before sending them to the mixed
output.

The *Volume* and *Gain* values are raw values. *Gain* is a volume multiplier
that is considered before the limiter. That is, it can make quiet moments a
little louder.

A **limiter** is a special effect that causes the volume to drop dynamically
when the input signal becomes too loud. It forces the input audio signal to not
cross a volume threshold. The limiter “looks ahead” at upcoming samples to
determine if one is too loud. If it finds one, it starts ramping down the volume
so that by the time the too-loud sample is ready to be played, it will be played
at an appropriate volume. Once things quiet down, the limiter is allowed to
restore the volume at a relaxation rate in decibels.

The *Range Multiplier* is a multiplier applied to the falloff ranges of all
sources for this specific listener. As an example, you might increase this value
if the player is using a character with enhanced hearing capabilities.

The *Interaural Time Difference Resolution* (ITD resolution) specifies the
resolution for computing the time it takes for sound to reach one ear versus the
other assuming the speed of sound in Earth’s atmosphere. A higher resolution may
increase the amount of sampling performed when voices are combined.

The *Listener Response Profile* allows for providing a custom spatial profile
which describes the filters applied to the audio sources. Such profiles can be
scriptable objects which derive from the abstract `ListenerProfileBuilder`
class. If no custom profile is applied, a default profile which provides basic
horizontally-spatialized audio is used. More info on creating these profile
builders in code can be found further down this page.

Lastly, you can specify a list of audio channel assets that the listener can
hear. Any audio source that uses any one of the channels in list will be heard
by the listener. Multiple listeners may be able to hear the same source. If
*Include Sources Without A Channel* is enabled, the internal default channel
will be added to the list.

### Audio Settings

The *Audio Settings* component allows for configuring system-level settings in
Myri. These settings must be applied to the *World Blackboard Entity* to have
any effect.

The master output settings specify the final output volume across all listeners
after they have been mixed together. The master output has one final limiter
stage, and these settings are also exposed here.

The mixing thread updates at a fixed interval independent of the application
framerate. In Myri’s world, these are referred to as “audio frames”. The audio
framerate is platform-dependent but is equivalent to: `sampleBufferSize /
sampleRate` in seconds. For a buffer size of 1024 sampled at 48 kHz, this
approximates to 20 milliseconds or 50 AFPS. Myri performs sampling synchronized
with the main thread rather than the mixing thread, and transmits the samples
over to the mixing thread with each update. If the main thread were to stall,
the mixing thread will eventually run out of samples and audio will cut out.

The *Safety Audio Frames* controls the number of extra audio frames Myri will
compute and send to the mixing thread per update. Higher values decrease the
chance of the mixing thread being starved but require more computational power.

*Log Warning If Buffers Are Starved* will instruct Myri to print a warning in
the console should the mixing thread run out of samples. The warnings will print
out info regarding the current audio frame and the audio frame last received.

Because the mixing thread updates independently of the main thread. It is
possible it may update while Myri is performing sampling. By the time Myri
finishes sampling, the new samples will be an audio frame old. If Myri’s ECS
system has the opportunity to update again before the mixing thread’s next
update, it will correct the issue. However, if the mixing thread updates again
first, it will skip ahead by a frame in the received sample set. This is usually
fine, but a problem arises when the received sample set contains the first
samples of a new one-shot. The first audio frame of that one-shot will be
omitted. This is significantly more likely to happen if the audio framerate is
higher than the main thread framerate.

In that case, *Audio Frames Per Update* can help mitigate this problem by
sending multiple audio frames in less frequent batches, effectively reducing the
audio framerate proportionally. This comes at a performance cost as well as less
“responsive” audio relative to the simulation, and some projects may prefer to
accept the occasional first audio frame drops and leave *Audio Frames Per
Update* at *1*.

For scenarios where the main thread framerate outpaces the audio framerate
slightly, but sampling consumes a large amount of time, it may instead be
preferred to begin sampling an audio frame or two early. Setting *Lookahead
Audio Frames* to a value of 1 or greater will accomplish this. While this
increases audio latency, it does not introduce a performance cost.

### Optimizing Clips for Performance

Every audio clip has a sample rate, typically measured in kHz. Common sample
rates are 44.1 kHz and 48 kHz. The audio output of the device may have a
different sample rate than the audio clip. When this happens, Myri needs to
“resample” the clip to compensate at runtime, which may have a measurable
performance impact for complex audio workloads. Therefore, it is recommended to
convert the clips’ sample rates to the target device’s sample rate if
performance is a concern. This can be done using the import settings for the
audio clip. The following image shows an audio clip with a sample rate of 44.1
kHz being converted to 48 kHz.

![](media/8986587ec983ff3f342baaf8990b8899.png)

## IComponentData

Myri’s entire API is exposed as Burst-friendly `IComponentData` that can be
interacted with inside jobs. Audio Sources use a modular set of components,
while listeners use a single `AudioListener` component.

### AudioSourceVolume

`AudioSourceVolume` is the most essential component for defining an audio
source. It has a single `volume` field, which is a raw value. To mute an audio
source, set the volume to zero. This will trigger a fast-path that avoids
sampling the source.

### AudioSourceClip

An `AudioSourceClip` describes an audio clip and its current playback state. If
the clip or looping properties change, the playback state will be reset. You can
also manually reset the playback state via `ResetPlaybackState()`, which you may
need to do if you copy the `AudioSourceClip` struct manually to another entity.
Entities spawned from prefabs will start in a reset state and do not require any
further intervention. For looping clips, `offsetIsBasedOnSpawn` toggles whether
the clip plays from the beginning, or whether it aligns to one of the clip’s
voices for ambience. Currently, `AudioSourceClip` is required for audio sources
to be heard.

### AudioClipBlob

The clip fields in the above components come in the form of a
`BlobAssetReference<AudioClipBlob>`. The most useful field for you is likely the
name, which is the original `name` of the clip asset when it was baked. The
`Length` of `loopedOffsets` is the number of unique voices for that clip when
played by looped sources.

Audio Clips have a full [Smart Blobber API](../Core/Smart%20Blobbers.md) which
you can use in your own bakers.

### AudioSourceDistanceFalloff

When `AudioSourceDistanceFalloff` is present, the audio source enables
spatialization for all listeners. It provides the three authoring values of
`innerRange`, `outerRange`, and `rangeFadeMargin`. These values use `half`
precision.

`AudioSourceDistanceFalloff` requires `WorldTransform` or `LocalToWorld` to be
present in order to have an effect.

### AudioSourceEmitterCone

When `AudioSourceEmitterCone` is present, and `AudioSourceDistanceFalloff` and
`WorldTransform`/`LocalToWorld` are also present, then the volume perceived by
the listener will be further filtered by the cone shape. The cone’s center axis
always faces the audio source’s forward direction.

### AudioSourceChannelID

When `AudioSourceChannelID` is present, it defines the channel that listeners
should be listening to if they want to hear this audio source. If absent or
`default`, then the audio source uses the internal default channel. At this
time, non-default instances must be baked, though they can be copied around at
runtime. To bake an instance, call `GetChannelID()` on the audio channel asset
from within a baker.

### AudioSourceSampleRateMultiplier

This component applies a multiplier to the sample rate of the clip. A value of
2f will result in the clip playing at double speed, and an octave higher in
pitch. Random values close to 1.0 can provide subtle variations to the sound,
which may be useful for commonly played sound effects. Be careful though,
because only sources with the exact same multiplier can be voice-combined.

Notably, changing this value while the clip is playing will NOT have the desired
effect, and will cause the playback state to jump around wildly. Set this value
when the entity spawns, and then leave it alone.

### AudioListener

The `AudioListener` component describes all parameters of the listener except
its transform and the audio source channels. A `WorldTransform` or
`LocalToWorld` must also be present for the listener to be valid.

The listener contains `volume`, `gain`, `limiterDBRelaxPerSecond`, and
`limiterLookaheadTime` values which describe the listener’s sound in the overall
mix. These can all be changed at any time, though artifacts may occur when
changing the `limiterLookaheadTime` while audio is playing.

The `rangeMultiplier` is the multiplier applied to the
`AudioSourceDistanceFalloff` values. Like those values, it represents the
multiplier in `half` precision.

The `itdResolution` (inter-aural time difference resolution) will be clamped to
[0, 15]. This value measures the number of steps from center to either ear that
audio can be delayed by. Higher values increase fidelity but decrease the
effectiveness of voice-combining, which comes with a performance cost. The
`listenerProfile` contains the metadata used to describe the spatialization
filtering.

Unlike audio sources, the internal listener state is stored in
`ICleanupComponentData`. The structural change commands are sent to
`SyncPointPlaybackSystem`. You typically do not have to worry about these
components.

### AudioListenerChannelID

Audio Listeners have a `DynamicBuffer<AudioListenerChannelID>` where each
element contains an `AudioSourceChannelID`. These represent the set of channels
that the listener can hear. A single element with just the default value will
hear the default channel. No elements will prevent the listener from hearing
anything.

### AudioSettings

The `AudioSettings` component exposes identical settings to its authoring
counterpart which can be modified and tuned at runtime. The component should
always exist on the `worldBlackboardEntity`. **Do not remove it!** Like all the
other Myri components, this component is read asynchronously from jobs.

## Creating a Custom Listener Response Profile

To create a custom profile, you may subclass `ListenerProfileBuilder` and
implement the `BuildProfile()` method. This is a `ScriptableObject`, so you can
expose settings and create custom editors for it if you wish. As an alternative,
you can implement the `IListenerProfileBuilder` on a struct type, but you are
responsible for applying the profile to an `AudioListener` component.

When building a profile, you create *filter channels*. These channels are
different from audio channel assets. A filter channel may be either a spatial
channel, or a direct channel. Each ear has its own set of channels.

A spatial channel is a rectangular area on a unit sphere which contains a unique
set of volume controls and filters for that area. When an audio source with
spatialization is heard by the listener, the direction from the listener to the
source is computed. If that direction lies within the spatial channel’s
rectangular area, that source will be processed by the channel. If a direction
does not lie within a channel, it will distribute the source’s audio signal to
multiple spatial channels using a weighting function. With the right filters and
volumes, this dynamic distribution can create dynamic and responsive listening
experiences.

*Weighting function details: The weighting function scans horizontally and
vertically around the unit circle until it finds a channel in each of the four
directions or fully wraps around. This creates a search rectangle, where all
channels inside or touching this rectangle receive weights proportional to their
distance from the direction vector. This approach, while moderately fast, has
some shortcomings. If no channel is detected in the horizontal and vertical
searches, then all channels will be evaluated rather than the nearest. I’m not
100% confident this is the best model, and would appreciate feedback.*

Direct filter channels are specific channels that hear non-spatialized audio
sources. All direct channels in a profile receive any given non-spatialized
audio source within the entire ECS context.

### Adding a Channel

To create a channel, call `AddSpatialChannel()` or `AddDirectChannel()`. These
return a `ChannelHandle` which can be used for adding filters to the channel.

The first argument in `AddSpatialChannel()`,
`minMaxHorizontalAngleInRadiansCounterClockwiseFromRight`, specifies the range
of the channel horizontally in radians, measured from the right ear
counter-clockwise looking from above. Values from -2π to 2π are accepted, but
the max value should always be greater than the min. The second argument
`minMaxVerticalAngleInRadians` specifies the range of the channel vertically in
radians, measured from the nose counter-clockwise looking from the right ear.
Much like the horizontal arguments, values from -2π to 2π are accepted, but the
max value should always be greater than the min.

The `volume` parameter applies a scalar volume control value relative to the
other channels. This is a raw scale factor without any limiting.

The last argument specifies which ear the channel should be associated with. Use
`true` for the right ear, and `false` for the left ear.

You can add up to 127 channels to a profile. There is no restriction as to how
these channels are distributed to the left and right ears.

### Adding a Filter

You can specify a filter using the `FrequencyFilter` struct type. The filter
types available are low pass, high pass, band pass, notch, low shelf, high
shelf, and bell. Each of these filters can be specified with a `cutoff`
frequency, a quality `q`, and a `gainInDecibels`.

Once you have described the filter, you can add it to a channel by calling
`AddFilterToChannel()`. The filters will be chained in the order you add them to
that specific channel.

*Filter details: The filters provided are based off the filters provided in the
DSPGraph samples. These filters share a unified state-variable filter (SVF)
kernel. The code has been slightly modified from the samples to improve the
flexibility of the code and remove unnecessary allocations. However, the actual
filtering logic remains unchanged.*

### Adding the Blob Asset

If attempting to bake listener profile blob assets for storage outside of a
listener component, use the `IBaker` extension method
`BuildAndRegisterListenerProfileBlob()`. This is not a smart blobber API.

*Q: Wait, why not?*

*A: Listener profiles are small and cheap to compute, and a smart blobber API
would have more overhead than needed.*

## Custom DSP

Custom DSP processing is currently not supported in Myri. The underlying
DSPGraph is not exposed in any way. A solution to this problem is coming soon in
the form of Effect Stacks, which will overhaul Myri and add a much more
customizable and controllable API surface.

In the meantime, you can procedurally generate audio clips at bake time using
the Smart Blobber API.
