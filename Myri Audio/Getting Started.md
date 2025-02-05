# Getting Started with Myri Audio

Myri in its current form is an audio solution which drew inspiration from the
Megacity 2019 demo. It currently supports simple use cases appropriate for game
jams, experiments, and maybe even some commercial projects. However, it lacks
the necessary control features required to deliver a final product in this
release. A major overhaul is planned for a future release which will lay the
foundation of something much more suitable.

With that said, I do believe that at the time of writing, Myri is still the
closest to a production-ready ECS audio solution for Unity Engine proper. Many
of the concepts discussed here will survive through the new overhaul. So I hope
you enjoy trying it out and feel free to send feedback!

## Setting up the Managed Driver

Many people encounter an issue with domain reload freezing when using Myri
out-of-the-box, forcing a restart of the editor to continue progress. If you
encounter this, you can go to the edit menu and select *Latios -\> Use Myri
Editor Managed Driver*. This setting is unique per computer per project.
Enabling this has two side-effects.

First, it requires a Unity Engine Audio Listener attached to the main camera.
This is the only time a Unity Engine audio component is ever used in Myri.

Second, the managed driver will always attach the Mono runtime and garbage
collector to the mixing thread. Normally, this doesn’t happen.

With these two alterations, the managed driver will typically have different
behavior in the editor when switching scenes or doing any other custom loading
compared to what will happen in a build.

## Playing Your First Sound

Create a new Unity Project with the Latios Framework installed. Then add the
*Latios Bootstrap* using the *Standard – Injection Workflow*.

Create a subscene, and add two empty Game Objects to it with their positions set
to the world origin (0, 0, 0). On one of the Game Objects, in the *Add
Component* *Menu*, select *Latios -\> Myri -\> Audio Listener (Myri)*. Leave all
the settings at default.

Next, import an audio source with the *Load Type* set to *Decompress On Load*
and *Preload Audio Data* checked. Then, on the non-listener Game Object, add a
*Latios -\> Myri -\> Audio Source (Myri)* component. Drag your clip into the
*Clip* field. Now enter play mode. You should hear your clip playing.

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
audio-emitting entity. When the listener is inside the *Inner Range*, it will be
heard at max volume. Myri uses inverse-square falloff, so at twice the *Inner
Range*, the audio source will be heard at quarter volume. The *Range Fade
Margin* provides a range inside the *Outer Range* where the volume will
interpolate to zero.

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

If *Use Cone* is checked, then the audio source receives an additional
directional attenuation factor. Listeners within the *Inner Angle* from the
source’s forward direction will not receive any attenuation. Listeners at the
*Outer Angle* will be attenuated by the *Outer Angle Volume*.

If *Auto Destroy on Finished* is checked and *Looping* is unchecked, the entity
containing the audio source will automatically be destroyed once the clip has
finished playing. This uses the [Auto-Destroy
Expirables](../Core/Auto-Destroy%20Expirables.md) mechanism.

### Audio Listener

The *Audio Listener* component provides all the settings for an entity that is
able to receive and spatially process sounds before sending them to the mixed
output. The *Interaural Time Difference Resolution* (ITD resolution) specifies
the resolution for computing the time it takes for sound to reach one ear versus
the other assuming the speed of sound in Earth’s atmosphere. A higher resolution
may increase the amount of sampling performed when voices are combined.

The *Listener Response Profile* allows for providing a custom spatial profile
which describes the filters applied to the audio sources. Such profiles can be
scriptable objects which derive from the abstract `ListenerProfileBuilder`
class. If no custom profile is applied, a default profile which provides basic
horizontally-spatialized audio is used. More info on creating these profile
builders in code can be found further down this page.

### Audio Settings

The *Audio Settings* component allows for configuring system-level settings in
Myri. These settings must be applied to the *World Blackboard Entity* to have
any effect.

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
preferred to begin sampling an audio frame earlier than normal. Setting
*Lookahead Audio Frames* to a value of 1 or greater will accomplish this. While
this increases audio latency, it does not introduce a performance cost.

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
interacted with inside jobs. Note that much of this API is likely to change in a
future release. The components will be divided up differently, but the
individual fields in the components will persist.

### AudioSourceLooped and AudioSourceOneShot

Looped and one-shot audio sources are handled separately in Myri. However, they
expose identical properties publicly. The `clip`, `volume`, `innerRange`,
`outerRange`, and `rangeFadeMargin` can all be configured freely.

These components also contain internal playback state. Copying a component will
copy its playback state. For looped sources, this means copying the unique
offset value. For one-shots, the actual playhead position will be copied. In
some cases this behavior is desirable, such as when working with component
arrays you may want to only modify the `volume`. Other times you may want to
reset the playback state. You can use the `ResetPlaybackState()` method to do
so. Changing the `clip` will also reset the state, but only if the new value is
different from the previous value.

A reset state will not be initialized until the next `AudioSystem` update, so
you do not have to call `ResetPlaybackState()` after copying instances that have
already been reset, or came from prefabs, or were constructed through code.

To mute an audio source, set the volume to zero. This will trigger a fast-path
that avoids sampling the source.

### AudioClipBlob

The clip fields in the above components come in the form of a
`BlobAssetReference<AudioClipBlob>`. The most useful field for you is likely the
name, which is the original `name` of the clip asset when it was baked. The
`Length` of `loopedOffsets` is the number of unique voices for that clip when
played by looped sources.

Audio Clips have a full [Smart Blobber API](../Core/Smart%20Blobbers.md) which
you can use in your own bakers.

### AudioListener

The `AudioListener` component is very simple, containing only three values. The
volume is the most useful. The `itdResolution` (inter-aural time difference
resolution) will be clamped to [0, 15]. This value measures the number of steps
from center to either ear that audio can be delayed by. Higher values increase
fidelity but decrease the effectiveness of voice-combining, which comes with a
performance cost. The `listenerProfile` contains the metadata used to describe
the spatialization filtering.

Unlike audio sources, the internal listener state is stored in
`ICleanupComponentData`. The structural change commands are sent to
`SyncPointPlaybackSystem`. You typically do not have to worry about these
components.

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

When building a profile, you create *channels*. A channel is a rectangular area
on a unit sphere which contains a unique set of volume controls and filters.
Each ear has its own set of channels. When an audio source is heard by the
listener, the direction from the listener to the source is computed. If that
direction lies within the channel’s rectangular area, that source will be
processed by the channel. If a direction does not lie within a channel, it will
distribute the source’s audio signal to multiple channels using a weighting
function. With the right filters and volumes, this dynamic distribution can
create dynamic and responsive listening experiences.

*Weighting function details: The weighting function scans horizontally and
vertically around the unit circle until it finds a channel in each of the four
directions or fully wraps around. This creates a search rectangle, where all
channels inside or touching this rectangle receive weights proportional to their
distance from the direction vector. This approach, while moderately fast, has
some shortcomings. If no channel is detected in the horizontal and vertical
searches, then all channels will be evaluated rather than the nearest. I’m not
100% confident this is the best model, and would appreciate feedback.*

### Adding a Channel

To create a channel, call `AddChannel()`. This returns a `ChannelHandle` which
can be used for adding filters to the channel.

The first argument `minMaxHorizontalAngleInRadiansCounterClockwiseFromRight`
specifies the range of the channel horizontally in radians, measured from the
right ear counter-clockwise looking from above. Values from -2π to 2π are
accepted, but the max value should always be greater than the min.

The second argument `minMaxVerticalAngleInRadians` specifies the range of the
channel vertically in radians, measured from the nose counter-clockwise looking
from the right ear. Much like the horizontal arguments, values from -2π to 2π
are accepted, but the max value should always be greater than the min.

The third argument `passthroughFraction` dictates the amount of the signal that
should bypass filters. Each channel has a filter route and a bypass route which
get mixed together in the final output. Each of these routes can have a custom
volume factor applied using `filterVolume` and `passthroughVolume` arguments
respectively.

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

## Custom DSP

Custom DSP processing is currently not supported in Myri. The underlying
DSPGraph is not exposed in any way. A solution to this problem is coming soon in
the form of Effect Stacks, which will overhaul Myri and add a much more
customizable and controllable API surface.

In the meantime, you can procedurally generate audio clips at bake time using
the Smart Blobber API.
