# Myri Audio

Myri Audio is a pure DOTS audio solution designed for handling a myriad of audio
sources. It leverages the C\# job system, the Burst compiler, and the Scriptable
Audio API. At the time of its latest release (when this page was last modified),
no official Unity Entities-based audio solution exists.

Check out the [Getting Started](Getting%20Started.md) page!

## Features

### Out-of-the-Box

Myri can play audio without requiring a single line of code. Simply add Myri
Audio Source components to your Game Objects and a Myri Audio Listener Authoring
to your camera transform and you are good to go. A brickwall limiter is applied
to the final audio result so your audio can remain distortion-free no matter how
chaotic your scene is.

### Spatialization

Myri processes sound in 3D space to provide the listener a sense of
directionality and immersion. While simpler than the cutting-edge techniques
used in VR applications, the model is customizable through a listener profile
API. From 2D panning to up to 127 distinct direction channels, Myri can be tuned
to meet your project’s needs.

The default profile produces a similar sound experience to the Megacity 2019
demo.

### Basic Mixing with Multiple Listeners

Myri uses a channel asset system that allows listeners to filter on specific
audio sources. Listeners are mixed together, with each listener having its own
limiter controls to shape the final output.

The easiest way to think about this is that not only does a listener have a
position in 3D space, but it also acts like a fader control in a mixing console.

### Voice Combining

Myri can detect when several audio sources are playing the same clip in unison
and combine them to save processing power and reduce cacophony of ambient
sounds, all without losing spatial or intensity information. This feature is in
stark contrast to Unity’s FMOD implementation where sources are omitted when too
many are played at once.

The mechanism Myri uses for this is inspired by the Megacity demo. But unlike
the demo, Myri can also combine one-shot sources.

Voice-combining is automatic and requires no effort from the user.

### Cone Emitters

Myri allows audio sources to define cone-shaped emitters. These can be used to
emulate directional audio emitters such as loudspeakers.

Cone emitters are fully modifiable at runtime.

### Speed-Up/Slow-Down Pitch Shifting

Myri allows clips to override the sample rate that clips can be played at.
Playing at a higher sample rate means that the clip plays faster, and the pitch
rises. This provides a simple mechanism to randomize the pitches of sound
effects.

### Compression

Myri supports both uncompressed and ADPCM compression for clips. More encoding
formats may be added in the future.

### ECS-Friendly Job Scheduling

Myri has no synchronous dependencies and will not force any jobs to complete
other than those it has scheduled in a previous update.

Most DSPGraph-based solutions schedule jobs from the mixer thread to perform
intensive sampling. These jobs must compete for resources with jobs scheduled
from the main thread, which can lead to frame-time inconsistencies for heavily
jobified projects. Myri does not schedule jobs from the mixer thread. Instead,
Myri schedules sampling jobs to execute during an ECS sync point, utilizing
worker threads that would otherwise be idle. This essentially makes Myri’s
sampling work “free”.

*Warning: While this approach reduces frame times, it requires more computation
effort. Consequently, it may not be well-suited for mobile platforms where
battery life is a concern. Effect Stacks will provide an alternative that may
perform better in these circumstances.*

### Simple, Easy, and Job-friendly

Myri aims to provide a simple, easy, and job-friendly API. Rather than using
shared components or managed components, Myri uses blob assets to store sample
data. You can easily query and assign clips to audio sources inside Bursted
jobs.

Audio sources automatically begin playing as soon as Myri detects them. For
those familiar with classical Unity terminology, audio sources always exhibit
“Play on Awake” behavior. This means playing a one-shot is as simple as
instantiating a prefab. You can even command Myri to automatically destroy the
entity when the clip is done playing.

### Extensible with Audio ECS

Myri contains a fully extensible ECS backend for processing audio built on Aux
ECS. This backend allows custom code to run inside the audio thread with Burst.
It provides a realtime allocator for the thread, as well as a scalable
thread-safe messaging API with the rest of the application. Blob assets from
subscenes are automatically kept alive even after the subscene unloads until the
Audio ECS backend has at least one update to process unload events. Scriptable
generators are supported.

## Known Issues

-   Myri may drop the first samples of a newly instantiated audio source if the
    audio framerate is faster than the simulation framerate. A configurable
    audio framerate multiplier exists to combat this, but issues may still arise
    during frame spikes.
-   Myri will only use up to `n` worker threads when performing sampling, where
    `n` is the sum of spatialization channels across all listeners. A default
    listener has six spatialization channels.
-   A high number of listeners can have a heavy load on the DSP thread and
    potentially overwhelm it due to the heavy processing required for each
    limiter.

## Potential Future Roadmap Items

-   Clip pausing
-   Hierarchical Mixing
