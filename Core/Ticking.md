# Ticking

Ticking is a game loop framework for fixed step simulation with interpolation.
It is entirely optional, and must be explicitly enabled for your project due to
its performance overhead. It serves as the foundational layer for networking
within the Latios Framework.

**Warning: Ticking does not have automatic Unity Transforms support. If you use
Unity Transfrms, you will need to write the integration yourself.**

## Background

These days, nearly every display can handle some kind of dynamic refresh rate.
And consequently, dynamic framerates are the norm. However, some games require
mechanics that don’t work well with dynamic framerates. This could be because of
physics, where simulation stability is of utmost importance. Or it could be
because of networking or replay mechanisms, where simulation reproducibility and
the ability to snapshot timepoints matters a lot.

These mechanisms require that the simulation be updated at a fixed rate, and
then be dynamically interpolated. There are three patterns to do this.

The first pattern involves simulating in fixed ticks up until the last full
step’s time has elapsed. Then, a partial tick is executed for interpolation. The
partial tick must then be reverted the next frame.

The second pattern also involves simulation in fixed ticks up until the last
full tick’s time has elapsed. However, rather than advancing a partial tick, it
simply calculates the ratio between the remaining partial step time and the
fixed tick time, and interpolates between the last two ticks into separate
components. This means that entities shown on screen are delayed by a fixed
tick’s worth of time.

The third pattern is the same as the second, except instead of the fixed ticks
lagging behind the elapsed time, one more tick is performed so that the final
tick time is ahead of the elapsed time, and then that final tick is reverted the
next frame.

Unity’s ECS networking solution uses a mixture of both the first pattern and
second pattern. Which pattern is used depends on the feature, without much rhyme
or reason.

The first pattern seems the most elegant at first, because interpolation comes
without having to write any new code. Or at least, that would be the case if you
didn’t need misprediction smoothing. Then it gets awkward, because frame history
and ticked history are two very different timelines, and both need to be
reasoned about. This naturally points towards the other two patterns which use
separate components for each timeline. The third pattern doesn’t have a tick’s
worth of input latency. Therefore, that is the pattern that is chosen for the
Latios Framework. **Ticking** is the set of systems, components, APIs, and tools
used to express this pattern efficiently.

Ticking is an optional set of Core systems which advance your simulation in
fixed-size steps independent of your framerate, and then interpolate the results
so that rendering stays smooth. Physics, replays, and networking all want a
simulation that behaves this way.

Unity already gives you `FixedStepSimulationSystemGroup`, which updates its
systems as many times as it takes to catch up to real time and does nothing
else. You do not get interpolation, so an entity sits still for a couple of
frames and then jumps. There is no tick number to name a moment by, and your
input arrives at framerate with nowhere sensible to put it, because as far as
the simulation is concerned, a tick either happened this frame or it didn’t.

Ticking fills in the rest. The loop keeps itself slightly ahead of your frame,
so that every frame lands between two simulated ticks and can interpolate
between them. Making that work means throwing ticks away and simulating them
again, which the whole design is built around. Your entities carry a second set
of components so that the simulation and the renderer stop fighting over the
same values, and a dedicated stage turns the tick results back into what the
renderer reads.

Installing ticking changes the shape of your frame quite a bit. If your game is
fine advancing at framerate, you don’t need any of this.

## Installation

**Warning: Ticking has major impacts on your game loop. If your game does not
need it, do not install it. Ticking has significant performance and
maintainability overhead.**

Ticking needs to be installed in both your baking bootstrap and your runtime
bootstrap. You do not need to install it in the editor bootstrap. In the runtime
bootstrap, there is only a *local* ticking mode, currently. This mode does not
support snapshots and rollback. Other modes will be added in later releases.

```csharp
public void InitializeBakingForAllWorlds(ref CustomBakingBootstrapContext context)
{
    CoreBakingBootstrap.InstallTicking(ref context);
    TransformsBakingBootstrap.InstallLatiosTransformsBakers(ref context);
    KinemationBakingBootstrap.InstallKinemation(ref context);
}

public bool Initialize(string defaultWorldName)
{
    var world                             = new LatiosWorld(defaultWorldName);
    World.DefaultGameObjectInjectionWorld = world;
    world.useExplicitSystemOrdering       = true;

    var systems = DefaultWorldInitialization.GetAllSystemTypeIndices(WorldSystemFilterFlags.Default);
    BootstrapTools.InjectUnitySystems(systems, world, world.simulationSystemGroup);

    CoreBootstrap.InstallLocalTicking(world, new CoreBootstrap.LocalTickingConfiguration
    {
        ticksPerSecond      = 60f,
        snapInputToNextTick = false
    });
    TransformsBootstrap.InstallTransforms(world);
    KinemationBootstrap.InstallKinemation(world);

    BootstrapTools.InjectRootSuperSystems(systems, world, world.simulationSystemGroup);
    // ...
}
```

**You must install ticking before any other modules.** Other modules may check
whether Ticking is installed and only install their Ticking-compatible pieces if
it is.

The runtime bootstrap installer requires a configuration. `ticksPerSecond` is
your tick rate. `snapInputToNextTick` specifies what happens when a frame spans
a tick boundary.

Let’s suppose frame X ended between ticks 500 and 501. And frame Y currently
updating will end between 501 and 502. When `snapInputToNextTick` is `true`,
frame Y will keep tick 501 as frame X left it, and only apply input to tick 502.
When `snapInputToNextTick` is `false`, frame Y will resimulate tick 501, merging
input with input acquired during frame X. Then frame Y will advance to tick 502.

The choice of `snapInputToNextTick` is a choice of game feel, and not of any
form of correctness.

## The Ticking Loop

Ticking operates in three distinct phases within a frame: Setup, Simulation, and
Interpolation.

Setup occurs inside `LatiosWorldSyncGroup`, which updates inside
`InitializationSystemGroup`. The system responsible is `TickedLocalSetupSystem`.
The system does all of its work on the main thread, and is responsible for
configuring most fields of the `TickingState` for the frame.

Simulation is driven by `TickedLocalSuperSystem`, which updates after
`FixedStepSimulationSystemGroup`, but still inside the `OrderFirst` bucket. It
updates one or more times in a row based on what `TickingState` prescribes. It
is also responsible for managing the world time for itself, and updating
`TickingState` for each tick. `TickedLocalSuperSystem` contains several
`SuperSystem`s designed for user code to inject into.

`TickedInterpolateSuperSystem` follows immediately after
`TickedLocalSuperSystem` and is responsible for interpolating values between
ticks for the frame. This system can be injected into directly.

`TickingState` is a component on the `worldBlackboardEntity`. It contains info
about both all the ticks in the frame, as well as the currently executing or
last executed tick. You can read from `TickingState`, but **do not write to
it.**

## Archetype Management

When ticking is enabled, the archetype of the entity is driven by two tag
components: `TickedEntityTag` and `TickingOnlyEntityTag`. Both of these can be
added by the *Ticked Entity* authoring component. The former specifies that
ticking components are required by the entity. The latter specifies that
interpolation components are not required by the entity. Note that these tags
are hints that systems react to. An entity without `TickedEntityTag` can still
have ticking components if a system determines it is needed for other entities
to function. An example of this is a non-ticked parent entity of a ticked child.
The parent needs ticking components to support the child’s ticked transform.

Archetypes are corrected inside `TickedArchetypeCorrectionSystemGroup`. This
system in two very different locations. It updates in TickedLocalSuperSystem
following the `tickedSyncPoint` playback, and it updates during
`EntitySceneOptimizations` when a closed subscene is importing. You are allowed
to inject your own systems into `TickedArchetypeCorrectionSystemGroup`. But due
to the `EntitySceneOptimizations` entry point, you cannot use `ILatiosApi` for
such systems.

### Ticking-Only Attribute Setup

Every ticking component requires a current state and a previous state. For a
simple ticking-only `IComponentData` or `IBufferElementData` you want to create
a “previous state” for, simply define a previous struct with a single field of
the current struct type. Then add `[TickedAutoAdd]` and `[TickedAutoPrevious]`
to the previous struct. Here’s what that might look like:

```csharp
public struct AiExploreState : IComponentData
{
    public float3 wanderPosition;
}

[TickedAutoPrevious(typeof(AiExploreState)), TickedAutoAdd(typeof(AiExploreState), true, false)]
public struct AiExploreStatePrevious : IComponentData
{
    public AiExploreState previous;
}
```

`[TickedAutoAdd]` has two `bool` parameters in addition to specifying what to
reference.

`copyData` refers to whether the contents of the current component should be
copied when the previous component is added. If this is `false`, the previous
component will be set to `default` instead. Which you use highly depends on
lifecycle. `[TickedAutoAdd]` processes entities with `Prefab` and `Disabled`
components. If an entity were to spawn with an initialized current component
wildly different from its prefab value, then `copyData` being set to `true`
could be problematic, whereas a `default` value could be recognized as an
uninitialized previous state.

`removeReferenceForTickingOnly` should be set to false in this scenario.

### Ticking and Interpolation Attribute Setup

If you have a component that requires interpolation, then you need to add
`[TickedAutoAdd]` to both the current and previous ticked components, pointing
them at the interpolated component. Then set `removeReferenceForTickingOnly` to
true. The previous ticked component still requires the `[TickedAutoPrevious]`
attribute.

### Manual and Mixed Setups

Using the attributes makes setup for the common use cases quick. However, for
complex relationships between components and entities, you might want to take
manual control. It is strongly recommended that you aggressively perform
structural changes on prefabs and disabled entities, and not wait for spawns.
This way, corrections only run once on subscene load.

If you require that an entity have components that don’t match what
`TickedEntityTag` and `TickingOnlyTag` specify, use an `OrderVersionFilter` on
your `EntityQuery` to avoid processing the same entity every frame.

## History Updates

`TickedUpdateHistorySuperSystem` is responsible for either advancing the tick,
or reverting it. This system updates after archetype corrections within the
ticking loop.

The `[TickedAutoPrevious]` attribute described above handles history
automatically when present. In fact, it doesn’t actually impact the archetype
correction at all. If you need to control history yourself, don’t use this
attribute.

To handle history yourself, you need to read the `TickingState` component, and
specifically, the `discardPreviousTick` property. When this property is `true`,
you must revert the current ticked component states by a full tick. This usually
means copying the previous states into the current states. When the property is
`false`, you do the opposite, copying the current states into the previous
states.

## Handling Input

First, make sure to read input into an `IComponentData` within
`SimulationSystemGroup` prior to TickedLocalSuperSystem.

In local ticking, there are three types of tick updates you need to handle. You
determine which scenario based on `TickingState`.

If `TickingState.isAdditiveInputTick` is `true`, that means the previous frame
applied input to the same tick that this current frame is now re-evaluating.
Multiple frame’s of input need to be applied to a single tick.
`TickingState.inputTickFraction` tells you how much of the tick was accounted
for by previous frame inputs. For some input types, you might use this to lerp
between the previous tick’s input value and the current frame’s input. In this
scenario, you may need to aggregate or count events for event-based input.

The second scenario is if `TickingState.isFirstInputTick` is `true`. This must
be checked after the previous check. In this case, the frame starts at nearly
the same time as a new tick. If you used `snapInputToNextTick` in your
configuration, then this scenario is much more common.

The third scenario occurs when `TickingState.isCatchupTick` is `true`. In this
case, the current tick is not the first tick being simulated during the frame.
And now input needs to stretch itself across multiple frames. The catch here is
that you want to sustain continuous input, but not repeat events.

In local ticking, it doesn’t currently matter if you handle these three
scenarios inside `TickedInputSuperSystem`, or `TickedSimulationSuperSystem`.
`TickedInputSuperSystem` updates after archetype corrections, but before history
updates, whereas `TickedSimulationSuperSystem` is the last to update in the
loop.

## Simulation

`TickedSimulationSuperSystem` is where all your ticked game logic should go.
`SystemAPI.Time` and `ILatiosApi` time will reflect the ticked time rather than
the frame time.

Like any gameplay system, you schedule jobs, move entities around, and process
events. The tricky part to watch out for is `tickedSyncPoint`. If the last tick
gets reverted on the next frame, `tickedSyncPoint` will drop all command buffers
instead of playing them back. This avoids duplicated instantiations for repeated
ticks. But it also means that if you try to use `tickedSyncPoint` from outside
`TickedSimulationSuperSystem`, those commands may not stick. In a similar vein,
using `syncPoint` from inside `TickedSimulationSuperSystem` can cause duplicate
commands across successive frames, which you probably don’t want either.

## Interpolation

After the ticking loop has completed, it is `TickedInterpolateSuperSystem`’s
turn. This only updates once per frame, and is intended to smooth any motion or
other values between ticks so that things don’t appear jittery on screen.

Unlike archetype correction and history, there’s no automatic mechanism to
interpolate arbitrary components. Usually various feature modules will provide a
default interpolator you can activate with a tag component. For example, QVVS
Transforms provides the `InterpolateLocalTransformTag`. At this point, previous
ticked component values and current ticked component values should represent the
final states of adjacent ticks, which can be interpolated.
`TickingState.finalTickFraction` provides the interpolation factor to use.

There may be situations where instead of interpolating within
`TickedInterpolateSuperSystem`, you want to interpolate directly inside a
`PresentationSystemGroup` system. In this case, you should use
`presentationTickFraction` instead, which accounts for whether n-1 rendering is
being used or not.
