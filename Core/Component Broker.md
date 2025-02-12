# Component Broker

Let’s suppose you have a job. In this job, you call a few methods. Each of those
methods call more methods. And those methods call even more methods. All of
these methods require accessing components and dynamic buffers on entities.
Passing `ComponentLookup` types to all these methods will balloon the parameter
list of these methods. You could define a struct that contains all these lookup
types, but that involves a lot of boilerplate, and it locks your jobs into that
set of types whenever you want to use those methods. You need something more
flexible than that. Something like an `EntityManager` where you can ask for
types generically from a universal interface.

What you need is `ComponentBroker`!

Most often, this type of problem comes up with scripting systems, whether they
be visual, data-driven, or perhaps Unika. In these environments, scripts may
want to access a wide variety of components at any given time, in a way that is
somewhat unpredictable.

## Creating a Component Broker with the Builder

In your system, you will need to define a `ComponentBroker` member variable,
similar to how you would cache the `LatiosWorldUnmanaged`.

```csharp
ComponentBroker componentBroker;
```

Next, inside `OnCreate()`, you initialize the `ComponentBroker` using a
`ComponentBrokerBuilder`. The constructor for a builder takes an allocator. This
is an allocator for the builder itself, and usually you will want to pass in
`Allocator.Temp`.

On the builder, you make a chain of `.With<>()` calls to specify the components
you wish to add to the broker. Each call can specify if the type will be given
read-only access or read-write access. If a type is specified multiple times,
read-write access overrules read-only access.

Similar to Fluent Queries, you can create your own extension methods to define a
set of types in bulk. This is required for any `IAspect` types, as Unity does
not maintain a type table for optional types.

Once you are finished specifying your types, you call `.Build()` and pass in the
`SystemState` as well as an allocator for the `ComponentBroker` instance.
Usually, you will allocate with `Allocator.Persistent`. Naturally, this means
you will also want to dispose your `ComponentBroker` in `OnDestroy()`.

```csharp
public void OnCreate(ref SystemState state)
{
    componentBroker = new ComponentBrokerBuilder(Allocator.Temp)
        .With<Latios.Psyshock.Collider>(false) // read-write component
        .With<Latios.Transforms.Parent>(true)  // read-only component
        .With<LinkedEntityGroup>(false)        // read-write buffer
        .With<Latios.Transforms.Child>(true)   // read-only buffer
        .With<SceneTag>(true)                  // shared components are always read-only and ignore your parameter
        .WithTransformAspect()                 // extension method for TransformAspect
        .With<Latios.Myri.AudioSourceLooped, Latios.Myri.AudioSourceOneShot>(false)         // you can specify up to 5 types in one With<>() call
        .Build(ref state, Allocator.Persistent);
}

public void OnDestroy(ref SystemState state) => componentBroker.Dispose();
```

`ComponentBroker` can support up to 127 read-write types, 128 read-only types,
and 16 shared component types simultaneously.

## Scheduling a Job

For simplicity, let’s suppose we have a job simply defined like this:

```csharp
struct Job : IJob
{
    public ComponentBroker componentBroker;

    public void Execute()
    {
        // ...
    }
}
```

We can’t simply schedule the job like normal, because `ComponentBroker` is a
massive struct. Instead, we have to use `ScheduleByRef()` semantics.

Also, we need to ensure we update the ComponentBroker with the latest
SystemState.

```csharp
public void OnUpdate(ref SystemState state)
{
    componentBroker.Update(ref state);

    var job = new Job
    {
        componentBroker = componentBroker
    };
    state.Dependency = job.ScheduleByRef(state.Dependency);
}
```

## Usage in Single-Threaded Jobs

`ComponentBroker` offers many APIs. In single-threaded jobs, these will be the
most common ones to use:

```csharp
public bool Exists(Entity entity)
public bool Has<T>(Entity entity)
public RefRO<T> GetRO<T>(Entity entity) where T : unmanaged, IComponentData
public RefRW<T> GetRW<T>(Entity entity) where T : unmanaged, IComponentData
public DynamicBuffer<T> GetBuffer<T>(Entity entity) where T : unmanaged, IBufferElementData
public EnabledRefRO<T> GetEnabledRO<T>(Entity entity) where T : unmanaged, IEnableableComponent
public EnabledRefRW<T> GetEnabledRW<T>(Entity entity) where T : unmanaged, IEnableableComponent
public bool TryGetSharedComponent<T>(Entity entity, out T sharedComponentOrDefault) where T : unmanaged, ISharedComponentData
```

`ComponentBroker` also provides extension methods for `ArchetypeChunk` to
replace type handles.

You can start using all these APIs right away in your job, without any initial
setup required.

**Warning:** If you use the single-threaded `IJobFor.Schedule()`, you will need
to call `RelaxSafetyForIJobFor()` in the job. This is a workaround for a Unity
job system issue. Not calling it may result in false safety errors.

## Usage in Parallel Jobs

In a parallel job, `ComponentBroker` will allow you to read components that were
granted read-only access just like in a single-threaded job. However, any
components that were granted read-write access will throw safety exceptions.

If you know you have exclusive access to an entity, you can set up the
`ComponentBroker` for safe write access using `SetupEntity()`. This method
requires you specify the entity via the Entity’s `ArchetypeChunk` and its index
in the chunk. If you have an `IJobChunk` job, this is trivial. For `IJobEntity`,
you can get the `ArchetypeChunk` by implementing the `IJobEntityChunkBeginEnd`
interface and caching the `ArchetypeChunk` in a job member during
`OnChunkBegin()`.

In a parallel job, for any given component type, you should either exclusively
use `GetEnabledRW<T>()` and `GetEnabledRO<T>()` or use
`TrySetComponentEnabled<T>()`. The latter is atomic, meaning you can use it on
any entity, not just the one used in `SetupEntity()`.

`ComponentBroker` provides `Get*IgnoreParallelSafety<T>()` APIs. These are
equivalent to adding a `[NativeDisableParallelForRestriction]` on the type. Use
them with care.
