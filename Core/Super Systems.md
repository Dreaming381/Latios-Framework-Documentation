# Super Systems

A Super System is a subclass of `ComponentSystemGroup` which provides
specialized functionality. Super Systems power the explicit system ordering
mechanism as well as the custom system update criteria mechanism.

## Explicit System Ordering

When using the Explicit System Ordering workflow, you must specify your system
order in the `CreateSystems()` method of your `SuperSystem` subclass. To do so,
simply make calls to `GetOrCreateAndAdd*System<T>()` in the order you want the
systems to run, where \* is replaced with either `Managed` or `Unmanaged`.

```csharp
public class ExampleSuperSystem : SuperSystem
{
    protected override void CreateSystems()
    {
        GetOrCreateAndAddManagedSystem<YourSubSystem1>();
        GetOrCreateAndAddUnmanagedSystem<YourISystem2>();
        GetOrCreateAndAddManagedSystem<AnotherSuperSystem>();

        //Automatically reuses the existing TransformSystemGroup to prevent ChangeFilter fighting.
        GetOrCreateAndAddManagedSystem<TransformSystemGroup>();

        //You can dynamically generate your systems here too!
        var assemblies = AppDomain.CurrentDomain.GetAssemblies();
        foreach (var assembly in assemblies)
        {
            foreach (var type in assembly.GetTypes())
            {
                if (typeof(ICustomInterface).IsAssignableFrom(type))
                {
                    GetOrCreateAndAddManagedSystem(typeof(YourGenericSystem<>).MakeGenericType(type));
                }
            }
        }

        //If you would like to sort your systems using attributes after explicitly creating them, you can call this here:
        SortSystemsUsingAttributes();
    }
}
```

Note that creating a `SuperSystem` is not sufficient to make your systems
update. You either need to create [RootSuperSystems](#root-super-systems) or
create and inject your top-most `SuperSystem` types using the `BootstrapTools`
API.

## Hierarchical Update Culling

You can cull updates of `SubSystem`s or `SuperSystem`s by overriding
`ShouldUpdateSystem()`.

When a `SuperSystem` is iterating through the list of systems to update, if it
detects a system is a `SuperSystem` or `SubSystem`, it will call
`ShouldUpdateSystem()` on the system and set the system’s `Enabled` property
appropriately. Setting the `Enabled` property triggers proper invocation and
propagation of `OnStartRunning()` and `OnStopRunning()`.

If a `SuperSystem` is disabled, its `OnUpdate()` will not be called and it will
not iterate through its children systems, potentially saving crucial main thread
milliseconds.

In the following example, the first frame `ShouldUpdateSystem()` returns false,
the three children systems will immediately have `OnStopRunning()` called on
them, but will not have `Update()` called on them. In the following frames where
`ShouldUpdateSystem()` returns false, the children systems will be left
untouched. This may lead to some non-obvious behavior if you are relying on
`OnStopRunning()` for a system that is a child of multiple `SuperSystem`s.

```csharp
public class BeastSuperSystem : SuperSystem
{
    EntityQuery m_query;
        
    protected override void CreateSystems()
    {
        GetOrCreateAndAddUnmanagedSystem<BeastHuntSystem>();
        GetOrCreateAndAddUnmanagedSystem<BeastEatSystem>();
        GetOrCreateAndAddUnmanagedSystem<BeastSleepSystem>();

        m_query = Fluent.WithAll<BeastTag>(true).Build();
    }

    public override bool ShouldUpdateSystem()
    {
        var scene = worldGlobalEntity.GetComponentData<CurrentScene>();
        return scene.current != "Beast Dungeon" || !m_query.IsEmptyIgnoreFilter;
    }
}
```

Note: You can use hierarchical update culling while also using the system
injection workflow. When doing so, simply do not call
`GetOrCreateAndAdd***System<T>()`.

### ShouldUpdateSystem() vs IRateManager

These are separate tools with separate intended purposes which can be used in
conjunction. `ShouldUpdateSystem()` is designed for turning off systems that are
not needed for a given scene or game mode or other context, whereas
`IRateManager` is designed for customizing the frequency the group of systems
update. For this reason, `ShouldUpdateSystem()` will affect the Enabled property
of itself and it and child systems will have `OnStartRunning()` and
`OnStopRunning()` called when the return value of `ShouldUpdateSystem()` differs
from its previous call. `IRateManager` will not do these things.

If you override `ShouldUpdateSystem()`, you must return an explicit `true` or
`false` value. Do not conditionally return the value provided by the base
`SuperSystem` type as this will simply return the value of the previous call to
`ShouldUpdateSystem()`.

You can use `SuperSystem` in Injection Workflow projects. However, you typically
would not add systems via `GetOrCreateAndAddSystem()` and instead rely on
`[UpdateInGroup]` attributes.

## Root Super Systems

Unlike regular `SuperSystem`s, `RootSuperSystem`s are designed to be injected
into traditional `ComponentSystemGroup`s. They serve as entry-points relative to
Unity and perhaps third-party systems from which explicit system ordering can
take over.

If a `RootSuperSystem` uses a custom `ShouldUpdateSystem()` implementation, how
that information is relayed in the Editor may differ from `SuperSystem`.
