# Rng and RngToolkit

`Rng` is a Burst-friendly type designed for providing deterministic random
numbers to parallel jobs. `Rng` utilizes `RngToolkit`, which is a collection of
functions for mapping `uint` values to other types and ranges.

## Using Rng

The simplest way to explain `Rng` is by example. So let’s assume we need to
generate an array of 1 million random positions and rotations each update.

The first step is to create our `Rng` instance as a private field in the system.

```csharp
Rng m_rng;
```

Next, we need to seed our instance. We could have seeded it when we declared it,
but for extra determinism we will seed it at the start of every scene. This seed
should be hardcoded, but we want to make sure it is different for every system.
To help detect copy and paste errors, a string can be passed in instead of a
`uint`. A good practice is to pass in the name of the system.

```csharp
public override void OnNewScene() => m_rng = new Rng("RandomizeTransformSystem");
```

We will need to define a job to perform our randomization. Our job is a parallel
job, so we need to ensure each entity gets unique random results. We do this by
calling `GetSequence()` which returns an `RngSequence` instance.

With this sequence, we can make consecutive calls to its random functions. The
API matches `Unity.Mathematics.Random`.

```csharp
struct Job : IJobFor
{
    public Rng rng;
    public NativeArray<RigidTransform> transforms;

    public void Execute(int i)
    {
        var random = rng.GetSequence(i);
        var position = random.NextFloat3Direction() * random.NextFloat(0f, 100f);
        var rotation = random.NextQuaternionRotation();
        transforms[i] = new RigidTransform(rotation, position);
    }
}
```

`Rng` does not allocate any memory. That means if we were to pass it to another
job and make the same calls, we would get the same results. To avoid this, we
call `Shuffle()` to get new sequences for corresponding indices. For
convenience, this method also returns itself after the update. We’ll assign this
returned result to job when scheduling.

```csharp
state.Dependency = new Job { rng = m_rng.Shuffle() }.ScheduleParallel(1000000, 32, state.Dependency);
```

We would call `Shuffle()` again for each new job we schedule in the same system,
so that each job gets a new set of random number sequences.

## Using SystemRng

When aiming for maximum performance using Burst-compiled `ISystem` and
`IJobEntity`, the `entityInQueryIndex` can cause a significant overhead when
nothing else uses it, as it must compute and base offset cache before running
the actual job.

For this reason, it is instead desired to use a sequence index per chunk rather
than per entity. `SystemRng` helps facilitate this behavior.

`SystemRng` is an `IComponentData` type designed to be added to the system
`Entity` in `OnCreate()` or `OnNewScene()` like so:

```csharp
public void OnNewScene(ref SystemState state) => state.InitSystemRng("AiSearchAndDestroyInitializePersonalitySystem"));
```

Next, your `IJobEntity` will also need to implement `IJobEntityChunkBeginEnd`,
and in the `OnChunkBegin()` method, you will need to invoke
`SystemRng.BeginChunk()`. After that, you invoke your random calls directly on
the `SystemRng` instance.

```csharp
[BurstCompile]
partial struct Job : IJobEntity, IJobEntityChunkBeginEnd
{
    public SystemRng rng;

    public bool OnChunkBegin(in ArchetypeChunk chunk, int unfilteredChunkIndex, bool useEnabledMask, in v128 chunkEnabledMask)
    {
        rng.BeginChunk(unfilteredChunkIndex);
        return true;
    }

    public void Execute(ref AiSearchAndDestroyPersonality personality, in AiSearchAndDestroyPersonalityInitializerValues initalizer)
    {
        personality.targetLeadDistance = rng.NextFloat(initalizer.targetLeadDistanceMinMax.x, initalizer.targetLeadDistanceMinMax.y);
    }

    public void OnChunkEnd(in ArchetypeChunk chunk, int unfilteredChunkIndex, bool useEnabledMask, in v128 chunkEnabledMask, bool chunkWasExecuted)
    {
    }
}
```

Finally, during scheduling, you can shuffle and assign the job `SystemRng` like
this:

```csharp
new Job
{
    rng = state.GetJobRng()
}.ScheduleParallel(m_query);
```

There is also `state.GetMainThreadRng()` for when you want to use `SystemRng` on
the main thread, such as for an idiomatic foreach.

## Rng Randomness

You might be wondering if this new random number generator is prone to repeating
sequences or other flaws, especially since it promises deterministic parallel
unique random numbers every update. Some of that will be answered by this [GDC
Presentation](https://www.youtube.com/watch?v=LWFzPP8ZbdU), which was the source
of inspiration for this solution.

In that presentation, Squirrel Eiserloh presents a noise function which takes
both a `seed` and a `position` as arguments. Repeatedly incrementing the
`position` generates a sequence of random values. However, using the previous
raw random value as the next `position` value generates a completely different
sequence. These two principles are combined to generate separate sequences
corresponding to unique indices. Technically, these sequences are all part of
one long sequence starting at different points. But the likelihood of the
sequences overlapping in the common use case is quite small, since typically
only a small quantity of random numbers is needed for a given entity in a given
job. Also, `Unity.Mathematics.Random` has the same limitation.

But so far, we have only discussed modifying the `position`. All of those
sequences completely change when the `seed` is modified. And that’s what
`Shuffle()` does. While it is possible that two systems could get their `Rng`
seeds aligned, this is rare and often either transient or easily remedied by
changing one of the initial seeds.

Regarding the actual noise function, Squirrel Eiserloh has since released an
improved generator using the same interface as the one he presented at GDC. This
improved version called “SquirrelNoise5” is what powers `Rng`.

## RngToolkit

Most random number generators used in games generate `uint` values. However,
`uint` values are not commonly useful in simulations. These raw values need to
be converted into indices, floating point values, directions, and even
quaternion rotations. The math to remap `uint` values to these more useful forms
is always the same, but it isn’t always trivial. That’s why `RngToolkit` exists.
It contains a bunch of methods to perform these remappings.

Hopefully it will save you time and trouble when building out an API for any
random number or noise generator you may have.
