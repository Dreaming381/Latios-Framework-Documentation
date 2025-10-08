# Changelog

All notable changes to this package will be documented in this file.

The format is based on [Keep a Changelog](http://keepachangelog.com/en/1.0.0/)
and this project adheres to [Semantic
Versioning](http://semver.org/spec/v2.0.0.html).

## [0.14.0] – 2025-10-?

Officially supports Entities [1.3.14]

### Added

-   *New Feature:* Added ADPCM compression, which can optionally be used for a
    clip with a codec selector in authoring or the `Codec` enum in baking
-   *New Feature:* Added various runtime inspector properties to
    `AudioSourceClip` providing info about the clip
-   Added the `clip` property to `AudioSourceClip` to allow setting or changing
    the clip at runtime
-   Added `AudioClipBlob.sampleCountPerChannel`, which replaces
    `samplesLeftOrMono.Length`
-   Added `DspTools` methods `AccumulateSignalNoisePower()` and
    `CalculateSignalToNoiseRatio()`, which allow comparing differences in audio
    signals such as for evaluating compression quality

### Changed

-   **Breaking:** `AudioClipBlob` fields `samplesLeftOrMono` and `samplesRight`
    have been replaced with `encodedSamples`, as the data may now be compressed
-   **Breaking:** Renamed `SampleUtilities` to `DspTools`

### Fixed

-   Fixed issue when *Use Falloff* was unchecked resulted in the audio source
    never playing

### Improved

-   Added DC offset compensation with a gradual release to make the waveform
    continuous and eliminate artifacts that resulted from changing an
    `AudioSourceVolume` or using `AudioSourceDistanceFalloff`

## [0.13.0] – 2025-7-6

Officially supports Entities [1.3.14]

### Added

-   *New Feature:* Added Audio Source Channels which allow listeners to listen
    to only a specific subset of sources
-   *New Feature:* Added a per-listener limiter to support proper level mixing
    between listeners
-   *New Feature:* Added limiter controls to each listener as well as the master
    output
-   *New Feature:* Added `AudioSourceSampleRateMultiplier` which can be used for
    generating subtle varieties to time and pitch of sources
-   Added `rangeMultiplier` to `AudioListener` which can be used to scale all
    spatial ranges of all sources the listener might listen to
-   Added `ListenerProfileBuilder` method `AddDirectChannel()`, which is a
    channel dedicated to sources that do not use spatialization
-   Made the `StateVariableFilter` public API, which allows performing DSP
    frequency filtering on individual samples passed in one-at-a-time

### Changed

-   **Breaking:** `AudioSourceOneShot` and `AudioSourceLooped` have been
    replaced with `AudioSourceClip`, `AudioSourceVolume`, and
    `AudioSourceDistanceFalloff`, where `AudioSourceClip` can represent either
    looping or non-looping clips, and `AudioSourceDistanceFalloff` is optional
    and enables spatialization when present
-   **Breaking:** `ListenerProfileBuilder` method `AddSpatialChannel()` only
    requests a single volume rather than differentiating between passthrough and
    filter volumes, as the passthrough mechanism has been removed
-   **Breaking:** `AudioClipOverrideBase` and `AudioClipOverrideRequest` have
    been replaced with `IAudioClipOverride` and executes within Myri’s baker
    context
-   Spatialization ranges now use the `half` type at runtime
-   The default `AudioSettings` `lookaheadAudioFrames` has been changed from 0
    to 1
-   A dedicated Myri ECS World is now created at runtime, which is used for
    internal purposes

### Fixed

-   Fixed an issue where voice combining would have an unintended volume scaling
    effect, making combined sources much louder than they should be
-   Fixed an issue where streaming blob assets could be unloaded while sampling
    was happening

### Improved

-   Simplified the underlying DSP Graph topology to reduce overhead
-   The Myri Burst Driver now outputs audio an audio frame sooner

### Removed

-   Removed various DSP Graph nodes, as these should have never been public to
    begin with

## [0.12.0] – 2025-2-23

Officially supports Entities [1.3.9]

### Added

-   Added Myri editor managed driver, which when enabled via the edit menu,
    bypasses the infinite domain reload editor bug many users encountered

### Improved

-   Changed Myri’s authoring component menu items to be more obvious they come
    from Myri
-   Added additional validation to audio clip baking
-   Improved audio clip baking performance by using Unity 6 `Span` APIs

## [0.10.3] – 2024-6-30

Officially supports Entities [1.2.1] – [1.2.3]

### Fixed

-   Fixed `AudioClipOverrideRequest` having the wrong baking type attribute,
    which resulted in exceptions during baking

## [0.10.2] – 2024-6-22

Officially supports Entities [1.2.1] – [1.2.3]

### Added

-   Added `AudioClipOverrideBase` and `AudioClipOverrideRequest` to streamline
    assignment of procedural audio clips to audio sources during baking

## [0.10.0] – 2024-5-27

Officially supports Entities [1.2.1]

This release only contains internal modernizations of the code

## [0.9.4] – 2024-3-16

Officially supports Entities [1.1.0-pre.3]

### Fixed

-   Fixed audio sources being played at half the volume compared to Unity
-   Fixed custom listener blob profiles double-applying attenuation factors to
    sources for specific channels

## [0.9.0] – 2024-1-20

Officially supports Entities [1.1.0-pre.3]

### Changed

-   `AudioSourceDestroyOneShotWhenFinished` is now an `IAutoDestroyExpirable`

## [0.8.4] – 2023-11-10

Officially supports Entities [1.0.16]

### Fixed

-   Fixed baking stereo clips with an odd number of samples per channel
-   Beginning with Unity 2022.3.13f1, editor crashes when baking audio clip
    assets in a subscene no longer occur

## [0.8.2] – 2023-10-8

Officially supports Entities [1.0.16]

### Fixed

-   Fixed baking multiple audio clips at once

## [0.8.0] – 2023-9-23

Officially supports Entities [1.0.16]

### Added

-   *New Feature*: Added Unity Transforms (Entities 1.0) support

### Improved

-   Myri no longer builds the DSPGraph or starts executing its runtime systems
    every frame right away, but instead waits until the first frame it detects
    any audio source or listener, as this seems to circumvent a Unity issue that
    would cause Unity to get stuck on domain reloads

## [0.7.0] – 2023-5-29

Officially supports Entities [1.0.10]

### Added

-   Added `Myri.Audio.DSP.SampleUtilities` methods for converting to and from
    decibel representation

### Changed

-   Myri uses QVVS Transforms for spatial audio

### Fixed

-   Fixed memory leaks on shutdown
-   Fixed warning message about DSP Graph leaked nodes which were caused by
    listeners not being cleaned up correctly on world destruction

### Improved

-   Improved XML documentation coverage

## [0.6.5] – 2023-2-18

Officially supports Entities [1.0.0 prerelease 15]

This release only contains internal modernizations of the code

## [0.6.1] – 2022-11-28

Officially supports Entities [1.0.0 prerelease 15]

This release only contains internal compatibility changes

## [0.6.0] – 2022-11-16

Officially supports Entities [1.0.0 experimental]

### Added

-   *New Feature:* Added Smart Blobber which allows for providing custom audio
    samples without an `AudioClip` object
-   Added `IListenerProfileBuilder` for building profiles with stack-allocated
    data
-   Added `Baker.BuildAndRegisterListenerProfileBlob()` extension method for
    creating `ListenerProfileBlob` blob assets

### Changed

-   **Breaking:** Authoring has been completely rewritten to use baking workflow

### Improved

-   `AudioSystem` is now a Burst-compiled ISystem

## [0.5.7] – 2022-8-28

Officially supports Entities [0.51.1]

### Changed

-   Explicitly applied Script Updater changes for hashmap renaming

## [0.5.6] – 2022-8-21

Officially supports Entities [0.50.1] – [0.51.1]

### Fixed

-   Fixed Editor Mode Game Object Conversion when destroying the default audio
    listener profile configurator

## [0.5.2] – 2022-7-3

Officially supports Entities [0.50.1]

### Fixed

-   Fixed stereo clip conversion crash introduced in 0.5.0

## [0.5.0] – 2022-6-13

Officially supports Entities [0.50.1]

### Added

-   *New Feature:* `AudioClipBlob` and `ListenerProfileBlob` are now generated
    using Smart Blobbers and generation can be requested from user authoring
    code
-   Myri is now installed via a `MyriBootstrap` installer, allowing users to
    exclude it if creating a `DSPGraph` instance breaks their project

### Changed

-   Renamed `IldProfileBlob` to `ListenerProfileBlob`

### Fixed

-   `AudioClipBlob.name` is now a `FixedString128Bytes` and can be used in Burst
-   Myri will now ignore null `AudioClipBlob` instances rather than crash

## [0.4.5] – 2022-3-20

Officially supports Entities [0.50.0]

### Changed

-   Updated DSPGraph to 0.1.0-preview.22.

### Fixed

-   Myri is now compatible with Burst 1.6.4.

## [0.4.4] – 2022-3-13

Officially supports Entities [0.17.0]

### Added

-   Added a new option for looping audio sources to begin playing from the
    beginning at spawn rather than a voice offset. While this drastically
    reduces voice-combining potential, it is useful for background music and
    similar use cases.

### Fixed

-   Fixed an issue where the ITD offset would cause arrays to be indexed with a
    negative value.

### Improved

-   Added XML documentation and tooltips for a significant amount of Myri.

## [0.4.3] – 2022-2-21

Officially supports Entities [0.17.0]

### Fixed

-   Fixed missing asset dependency declarations during conversion for subscenes.

## [0.4.0] – 2021-8-9

Officially supports Entities [0.17.0]

### Added

-   Added AudioSettings.lookaheadAudioFrames which can be used to correct for
    when sampling regularly takes too long to deliver to the audio thread.

### Changed

-   **Breaking:** The concept of “subframes” has been removed. Audio sources now
    synchronize with the actual audio frame definition. In audioSettings,
    audioFramesPerUpdate now represents how many frames are expected to be
    played before the next buffer is consumed, effectively replicating the old
    subframe behavior. A new variable named safetyAudioFrames represents how
    many additional audio frames to sample in each update to prevent starvation.

### Improved

-   Myri is now more robust to AudioSettings changes while clips are playing.
-   *New Feature:* Myri now has a brickwall limiter at the end of its DSP chain
    on the audio thread. The current implementation may consume measurable
    resources on the audio thread and potentially lead to performance
    regressions. However, this limiter removes the distortion artifacts when too
    many loud audio sources play at once. Balancing the volume of audio sources
    is much less important now, and the overall out-of-the-box quality has
    greatly improved.

## [0.3.3] – 2021-6-19

Officially supports Entities [0.17.0]

### Fixed

-   Fixed an issue where looping sources with a sample rate different from the
    output sample rate would begin sampling from the wrong location each update.

## [0.3.2] – 2021-5-25

Officially supports Entities [0.17.0]

### Fixed

-   Renamed some files so that Unity would not complain about them matching the
    names of GameObject components.

## [0.3.0] – 2021-3-4

Officially supports Entities [0.17.0]

### This is the first release of *Myri Audio*.
