# Latios Framework Core

Latios.Core offers a toolbox of extensions to the Unity.Entities package. It
does not abstract away ECS nor provide an alternative ECS. The other modules in
this framework all depend on Core.

Check out the [Getting Started](Getting%20Started.md) page!

## Features

### Bootstrap Tools

If you need more customization than what the `DefaultWorldInitialization`
provides, you’ll find you have to do nearly everything yourself. The
`BootstrapTools` class is here to help you out!

-   Inject systems based on namespaces
-   Build partial hierarchies
-   Build a `playerloop` with custom preset loops (like one with classical
    `FixedUpdate` or one with Rendering before Simulation)

Also, there’s an `ICustomBakingBootstrap` interface which allows you to
customize baking worlds and enable or disable bakers. Do you want Psyshock to
bake colliders, or are you using Unity Physics exclusively? Should Entities
Graphics bake skinned meshes, or should Kinemation? It used to be a pain to
implement these decisions. But now you have full control!

And lastly, there’s `ICustomEditorBootstrap`, so you can customize the Editor
World too. This is especially useful for custom tooling that requires
interactive previews.

See more: [Customizing the Bootstraps](Customizing%20the%20Bootstraps.md)

### Explicit Top-Down Hierarchical System Ordering

Do the `[UpdateBefore/After]` attributes confuse you? Would you rather just see
the systems ordered explicitly in your code within the group as they do in the
editor windows? Would you like to decouple the systems so that they don’t
reference each other? Would you like to switch back and forth between two system
orderings?

If you answered “yes” to any of those, then you definitely want to try the new
way to order systems in this framework. You can specify the system order
explicitly using the `GetOrCreateAndAdd` API in `SuperSystem.CreateSystems()`.
It is opt-in, meaning you can still use the injection approach if you prefer
that.

See more: [Super Systems](Super%20Systems.md)

### Conditional System Updates

Unity does this thing where it tries to look at your Entity Queries and decide
if your system should update or not. While it’s certainly cute that Unity cares
about performance, you as the programmer can make much better decisions. Turn on
your own logic by overriding `ShouldUpdateSystem()` on a `SubSystem`
(`SystemBase`) or `ISystemShouldUpdate` (for `ISystem`).

You can also combine this with Unity’s logic if it is not interfering but you
also want to further constrain the system to a specific scene or the value of a
specific component on a specific entity.

You can also apply this logic to a `SuperSystem` (`ComponentSystemGroup`) to
enable or disable an entire group of systems. This can help improve main thread
performance.

See more: [Super Systems](Super%20Systems.md)

### Blackboard Entities

Do you ever get confused or frustrated by Unity’s singletons? Isn’t it awful
when you enter play mode and you get an error about a missing singleton? Then
you create an instance inside `OnCreate()` with reasonable defaults, only for
that to break when the designer finally starts using the authoring component to
customize the defaults. Why?!

And aren’t singletons annoying to access outside a system because they need an
`EntityQuery`? Also, when are you supposed to access them with read-only vs
read-write permissions? And does access them sync jobs or not? Why are the rules
different from other entities? Why are the rules different for buffers? Why do
singletons use so much memory for one entity? Why do they always trigger system
updates when using `[RequireMatchingQueriesForUpdate]`?

I despise Unity’s singletons. They are frustrating because someone at Unity
decided to hack them to solve common problems, rather than design proper
solutions for those problems.

Blackboard entities are one half of a proper solution. They derive from the
observation that when developers make singletons, they don’t actually want to
enforce exactly one instance of something. Really, they just want one particular
governing instance that everything else knows how to find. The framework
includes two easy-to-find entities. The `worldBlackboardEntity` lives with the
`World`, while the `sceneBlackboardEntity` dies and respawns when the scene
changes.

Blackboard entities come with streamlined APIs, as well as an authoring workflow
that allows for entity merging with custom override rules. Many modules in the
framework use them to expose settings or communicate between systems. And there
is plenty of room for you to use them for your own purposes too.

See more: [Blackboard Entities](Blackboard%20Entities.md) and [Blackboard
Entities vs Singletons](Blackboard%20Entities%20Vs%20Singletons.md)

### Collection Components

Why is storing collections on entities only properly supported for singletons?
Why are singleton rules different just to store collections?

The second half of my singleton replacement solution is Collection Components.
These are structs that implement `ICollectionComponent`. You can put them on any
entity, including blackboard entities. And they come with some really nice
features of automatically managing `SystemState.Dependency` and receiving
`TryDispose()` callbacks.

Just don’t try to iterate multiple `ICollectionComponent` instances across
multiple entities in a job. Unity’s job safety system can’t handle that.

See more: [Collection and Managed Struct
Components](Collection%20and%20Managed%20Struct%20Components.md)

### Scene Management

Remember the good-old-days in `MonoBehaviour` land where by default when you
loaded a new scene, all the `GameObject`s of the old scene went away for you?
That was kinda nice, wasn’t it? It’s here too!

The rule is simple. If you want stuff to disappear, use scenes. If you want
stuff to be additive, use subscenes.

You can request a new scene using the `RequestLoadScene` component, and you can
get useful scene info from the `CurrentScene` component attached to the
`worldBlackboardEntity`. All subscenes loaded are forced to load synchronously
by default, ensuring that essential entities are ready right away.

If you want an entity to stick around (besides the `worldBlackboardEntity` which
always sticks around), you can add the `DontDestroyOnSceneChangeTag`.

Now you can build your Mario Party clone, your multi-track racer, or your fancy
5-scene credits system in ECS with the ease you had in `MonoBehaviour`s. Oh, and
having most scenes be almost exclusively `MonoBehaviour`s while a couple of
scenes use ECS is totally a thing you can do, especially since you can check
`CurrentScene` inside `ShouldUpdateSystem`. For all you out there trying to
shoehorn ECS into your Mono game, shoehorn no more!

*This feature is not enabled by default for any bootstrap except the
DreamingBootstrap. It may require tweaks to work correctly with systems that
rely on singletons, such as Unity Physics.*

See more: [Scene Management](Scene%20Management.md)

### Fluent Queries

Unity’s `QueryBuilder()` API is very streamlined, but it isn’t very smart. If
two extension methods happen to repeat the same component, they explode.

Fluent Queries are a smart version of that API that can resolve duplicates
including reading vs writing conflicts and enabled state filtering differences.
The rules are explicit, well-documented, and designed to be intuitive when
different pieces of code all try to contribute to a single `EntityQuery`.

You begin a Fluent chain by calling `Fluent()` on a `SystemState`. To get the
resulting `EntityQuery`, call `Build()`.

See more: [Fluent Queries](Fluent%20Queries.md)

### Smart Sync Point and Custom Command Buffers

`EntityCommandBuffer` is a powerful tool, but it has some limitations.

Its biggest flaw is that it plays back commands one-by-one, which can be quite
slow and doesn’t take advantage of cache efficiencies. In many cases, your ECB
is only doing one kind of operation, such as destroying entities, or
instantiating them along with a fixed set of components.

Enter custom command buffers!

These command buffers focus on specific operations, and offer improved playback
performance as a result. `EnableCommandBuffer` and `DisableCommandBuffer` can
optimally enable and disable entities and their `LinkedEntityGroup` buffers.
`DestroyCommandBuffer` can destroy entities en masse. `InstantiateCommandBuffer`
can instantiate entities and initialize several components in one go. And
`AddComponentsCommandBuffer` is especially well-suited for adding cleanup
components, since it can detect destroyed entities and create temporary entities
for cleanup systems to process. There’s even a `CustomCommandBuffer` that lets
you define your own commands.

All of these command buffers can be played back by the `SyncPointPlaybackSystem`
(which can play back `EntityCommandBuffers` too). You can fetch this using
`LatiosWorldUnmanaged.SyncPoint.` And you don’t even have to invoke
`AddJobHandleForProducer()` or touch a singleton. Dependency management is fully
automatic. All that boilerplate is gone. As the title says, this sync point is
smart!

You can also play back these buffers manually.

See more: [Custom Command Buffers and
SyncPointPlaybackSystem](Custom%20Command%20Buffers%20and%20SyncPointPlaybackSystem.md)

### ILatiosApi, IInjectable, and IJobEach

Do you find Unity’s source-generated code unpredictable, inflexible, or still
too much boilerplate? The Latios Framework has a whole stack of source
generators and APIs to solve this problem.

The foundation is `ILatiosApi`, which your `ISystems` can implement to unlock an
alternative to `SystemAPI` for accessing `ComponentLookup` and similar. But
unlike `SystemAPI`, it allows you to define your own aggregate types. And it
does it without the source generators rewriting your entire system, making for a
better debugging experience.

But why even bother fetching handles? Why not just ask your job struct to fetch
them itself? That’s what `IInjectable` does. Simply call `Inject()` on it from
inside an `ILatiosApi` system and it will automatically populate fields
decorated with the `[Inject]` attribute.

Lastly, `IJobEach` is essentially `IJobEntity`, but with a few tweaks and
upgrades. Gone is the confusion around `EnabledRef`, as enabled state filtering
is opt-in rather opt-out. There’s no need to repeat yourself with an
`EntityQuery`, as you can actually obtain the auto-generated query to get the
entity count. And for extra extensibility, you can even define your own custom
parameters for the `Execute()` method.

See more: ILatiosApi

### EntityWith\<T\> and EntityWithBuffer\<T\>

Have you ever found an `Entity` reference in a component and wondered what you
are supposed to do with it? Do you instantiate it? Do you manipulate it? Do you
read from it? Maybe the name might give you a clue, but we all know naming
things is hard.

So instead, use `EntityWith<T>` and `EntityWithBuffer<T>`! They work just like
normal `Entity` references, except you can gather additional context about them.
An `EntityWith<Prefab>` should probably be instantiated. An
`EntityWith<Disabled>` needs to be enabled at the right moment. An
`EntityWith<LocalToWorld>` is a transform to spawn things at or attach things
to.

### Smart Blobbers

Blob Asset Baking is tough to get right, especially if you want to cache
expensive computation and generate blobs in parallel via baking systems. And
what if multiple bakers want to leverage this caching and parallel baking, but
do something custom with the results?

Smart Blobbers solve this problem by establishing patterns and adding guardrails
around baking systems, so that you can bake your blobs in parallel safely. They
provide a built-in mechanism for Bakers to request blobs to be built, and those
requests can later be resolved into real `BlobAssetReference<>` values. Smart
Bakers and Smart Post-Processors provide a mechanism to propagate bake context
beyond a Baker’s scope so that blob assets can be resolved correctly without
having to write custom baking systems.

The design outlines a pattern you can follow which allows you to extend the
Smart Blobber mechanisms to your own blob types. The process is thoroughly
documented and all APIs contain detailed XML documentation. You can obtain fully
parallel and Burst-compiled blob asset generation without having to worry about
Unity’s `BlobAssetStore` caching and deduplication.

See more: [Smart Blobbers](Smart%20Blobbers.md)

### UnsafeParallelBlockList and UnsafeIndexedBlockList

These containers are really unsafe. They were originally only meant for internal
purposes. But here they are on this page. Why?

They are fast! Really fast!

So fast that people were finding ways to use them anyways, whether that be
copying the code or modifying the package. Now they are public API so people
don’t have to do those workarounds anymore.

Just be warned. They really truly are a gun eager to put a bullet through your
foot.

### DynamicHashMap and DynamicMultiList

`DynamicHashMap` is a wrapper around a `DynamicBuffer` that provides
hashmap-like functionality. Unlike other implementations, this implementation
can correctly handle serialization of Entity and blob asset references. Although
explicit API calls must be made if a key’s hashcode changes before and after
serialization.

Similarly, `DynamicMultiList` allows embedding multiple resizeable lists inside
a `DynamicBuffer`.

See more: [DynamicHashmap](Dynamic%20Hashmap.md)

### Baking Interface Methods

You know how with Game Objects you can call `GetComponent<ISomeInterface>()` but
you can’t with `IBaker.GetComponent<ISomeInterface>()`?

I fixed that for ya.

### ComponentBroker and TempQuery

Do you ever have function pointers or maybe a massive collection of static or
helper methods and wanted to use `EntityManager` or evaluate entity queries in
those methods while in a job?

`ComponentBroker` provides a mechanism to specify a number of component types to
send to a job, and then while in a job, you can access them via generic methods.
You don’t have to pass a bunch of different type handles or lookups to your
methods. You can instead pass around the `ComponentBroker` by `ref`, keeping
your method arguments clean. Meanwhile, `TempQuery` allows you to evaluate an
entity query on-the-spot in the job and iterate over chunks and entities. It
isn’t as performant as cached entity queries created on the main thread, but it
can still be handy.

These features are especially useful for mods or scripting solutions such as
Unika.

See more: [Component Broker](Component%20Broker.md) and [Temp
Queries](Temp%20Queries.md)

### Live Baking Tracking

Live baking is powerful. But sometimes overwriting runtime components with
newly-baked values can break assumptions in your runtime systems. If you need
extra logic to handle these edge-cases, then `LiveBakedTag` can help you
identify entities of concern. And `BeforeLiveBakingSuperSystem` and
`AfterLiveBakingSuperSystem` will help you only run that extra logic when you
actually need it.

See more: [Live Baking Tracking](Live%20Baking%20Tracking.md)

### VPtr

VPtr is a source-generation solution for performing Burst-compatible
struct-based polymorphism using pointers to struct data. You can combine it with
`NativeStream` to perform polymorphic events, type-erase data down to the
interface level for complex method chains, or maintain untyped storage with the
option for callbacks for specific things.

`BlobVPtr` lets you bake polymorphic data into blobs.

See more: [VPtr](VPtr.md)

### Ticking

There are many types of games where the simulation rate needs to be held steady,
independent of the framerate. There’s a lot involved to allow reverting ticks
and interpolating the results. The optional Ticking APIs provide a foundation to
do those things efficiently.

See more: Ticking

### Extensions, Exposed, and ECS Cleanups

Sometimes Unity is missing API for no good reason other than DOTS still being
under development. And sometimes, I need this missing API. Sometimes this can be
fixed with extension methods. Sometimes this requires extending the package
directly using asmrefs. The former can be found in the Utilities folder, and the
latter shows up in the `Unity.Entities.Exposed` namespace.

There are also situations where Unity’s package just does really dumb things
that you probably weren’t aware of. For example, all prefabs will allocate on
the heap when instantiated because of an included `LinkedEntityGroup`, even if
they have no children. Or root component system groups might inject into
themselves if you construct them early.

I keep finding ways to mitigate bugs and missing APIs, and most of that ends up
in Core.

## Known Issues

-   `IManagedComponent` and `ICollectionComponent` are not true components. They
    instead rely on a source-generated `ExistComponent` to signify presence with
    a specific entity.
-   Compile errors are generated when using .Net Framework. Use .Net Standard.
-   IL2CPP may require the IL2CPP Code Generation to use the “Faster (smaller)
    builds” option.
-   Blackboard Entities do not retain blob asset reference counts.

## Potential Future Roadmap Items

-   `ThreadStackAllocator` `AllocatorManager` support
-   Entity create/change/destroy pipeline
-   Subscene loading world bootstraps
-   Bootstrap Profiles
    -   Allow multiple bootstraps per project for samples and tests
-   More custom command buffer types
-   Improved collection components
    -   Default initialization interface
    -   Get as ref
    -   Inspectors
-   Reflection-free improvements
