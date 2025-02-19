# Latios Framework Core

Latios.Core offers a toolbox of extensions to the Unity.Entities package. It
does not abstract away ECS nor provide an alternative ECS. The other modules in
this framework all depend on Core.

Check out the [Getting Started](Getting%20Started.md) page!

## Features

### Bootstrap Tools

The bootstrap API has improved quite a bit over time, but it still doesn’t let
you really customize the injection pipeline. That’s what the `BootstrapTools`
class is here for!

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
It is opt-in so if you prefer the injection approach you can do things that way
too.

See more: [Super Systems](Super%20Systems.md)

### Conditional System Updates

Unity does this thing where it tries to look at your Entity Queries and decide
if your system should update or not. While it’s certainly cute that Unity cares
about performance, you as the programmer can make much better decisions. Turn on
your own logic by overriding `ShouldUpdateSystem()` on a `SubSystem`
(`SystemBase`) or `ISystemShouldUpdate` (for `ISystem`).

You can also combine this with Unity’s logic if it is not interfering but you
also want to further constrain the system to a specific scene or something. I do
this a lot.

You can also apply this logic to a `SuperSystem` (`ComponentSystemGroup`) to
enable or disable an entire group of systems. This can help improve main thread
performance.

See more: [Super Systems](Super%20Systems.md)

### Blackboard Entities

I despise Unity’s singletons. They are a hack with way too much baggage.

Unity’s solution to singletons is to create an `EntityQuery` every time you want
a new one and also make every singleton component live in its own 16 kB mansion.
These singletons are so spoiled!

Well I am done spoiling singletons, so I asked myself:

Why do people write singletons? Is it because they only want one of something,
or is it really because they want most logic to find the same shared instance?

Well for me, it is the latter.

My solution is blackboard entities. There are two of them: world and scene. The
`worldBlackboardEntity` lives as long as the world does. The
`sceneBlackboardEntity` dies and respawns when the scene changes. These two each
get a 16 kB chunk for all of their components, and they even have a special
convenient API for getting and setting components directly on them.

But the best part is that there’s no `EntityQuery` associated with them, so if
you want to make a hundred backups of these entities to track your state over
time, or have a component represent the “Active” option from a pool of entities
containing “viable” options, well you can do those things here. Or maybe you
just like using `[RequireMatchingQueriesForUpdate]` like me.

Wait, no. That wasn’t the best part. The best part is the authoring workflow!
Simply attach a `BlackboardEntityData` component to automatically have all the
components merged into one of the two blackboard entities of your choice. You
can even set a merging strategy for them. This also works at runtime, so you can
do cool stuff like instantiate a new settings override from a prefab.

Regardless of whether you use the authoring tools, feel free to dump components
onto these entities. The `SceneManagerSystem `and Myri’s `AudioSystem `use the
`worldBlackboardEntity` to expose status and settings. And Kinemation uses it
heavily for internal communication and for exposing camera culling parameters to
the custom culling API.

Blackboard entities can be accessed as properties of `LatiosWorld`,
`LatiosWorldUnmanaged`, `SubSystem`, and `SuperSystem`.

See more: [Blackboard Entities](Blackboard%20Entities.md) and [Blackboard
Entities vs Singletons](Blackboard%20Entities%20Vs%20Singletons.md)

### Collection Components

Why in the world does Unity think that collections only belong on systems or
singletons? Did they not watch the Overwatch ECS talks?

I don’t like singletons, and I also don’t like tracked collections on entities
being exclusive to special entities, so I made my own solution for storing and
managing collections on entities. Collection components are structs that
implement `ICollectionComponent`.

**Warning: Collection components are not real components. The real component is
the** `ExistComponent` **nested type defined by source generators.**

You can query for the `ExistComponent` in your own code to find entities with a
collection component. You can then access the collection component using the
`LatiosWorldUnmanaged` methods.

Collection components have this nice feature of automatically updating their
dependencies if you use them in a system.

See more: [Collection and Managed Struct
Components](Collection%20and%20Managed%20Struct%20Components.md)

### Scene Management

Remember the good-old-days in `MonoBehaviour` land where by default when you
loaded a new scene, all the `GameObject`s of the old scene went away for you?
That was kinda nice, wasn’t it? It’s here too!

The rule is simple. If you want stuff to disappear, use scenes. If you want
stuff to be additive, use subscenes. And yes, having multiple scenes each with a
bunch of subscenes is totally supported and works exactly as you would expect it
to *(hopefully)*.

You can request a new scene using the `RequestLoadScene` component, and you can
get useful scene info from the `CurrentScene` component attached to the
`worldBlackboardEntity`. All subscenes loaded are forced to load synchronously
by default, ensuring that settings entities are present right away.

If you want an entity to stick around (besides the `worldBlackboardEntity` which
always sticks around), you can add the `DontDestroyOnSceneChangeTag`.

Now you can build your Mario Party clone, your multi-track racer, or your fancy
5-scene credits system in ECS with the ease you had in `MonoBehaviour`s. Oh, and
having most scenes be almost exclusively `MonoBehaviour`s while a couple of
scenes use ECS is totally a thing you can do, especially since you can check
`CurrentScene` inside `ShouldUpdateSystem`. For all you out there trying to
shoehorn ECS into your Mono game, shoehorn no more!

*This feature is not enabled by default for any bootstrap except the
DreamingBootstrap. It is also not compatible with NetCode and may require tweaks
to work correctly with systems that rely on singletons, such as Unity Physics.*

See more: [Scene Management](Scene%20Management.md)

### Math

What framework would be complete without some math helpers? Not this one.
Overly-used algorithms and some SIMD stuff are here. Help yourself!

See more: [Math](Math.md)

### Fluent Queries

Fluent syntax for expressing EntityQueries was a big improvement. However, every
iteration so far has lacked a good way to extend it. This implementation not
only provides clean Fluent syntax for building EntityQueries, but it is also
extensible so that library authors can write patch methods for their
dependencies.

Fluent Queries have a concept of “weak” requests, or requests that can be
overridden by other expressions. For example, an extension method may request a
read-only `Collider`. If a `Collider` request already exists (read-only or
read-write), the read-only request will be ignored.

There are similar mechanisms for handling all the various combinations including
enabled states. The rules are explicit, well-documented, and designed to be
intuitive when different pieces of code all try to contribute to a single
`EntityQuery`.

You begin a Fluent chain using the `Fluent` property on `SubSystem` and
`SuperSystem` or by invoking `Fluent()` on an `EntityManager` or `SystemState`.
To get the resulting `EntityQuery`, call `Build()`.

See more: [Fluent Queries](Fluent%20Queries.md)

### Smart Sync Point and Custom Command Buffers

`EntityCommandBuffer` is a powerful tool, but it has some limitations.

The biggest is that it plays back commands one-by-one, which can be quite slow
and doesn’t take advantage of cache efficiencies. In many cases, your ECB is
only doing one kind of operation, such as destroying entities, or instantiating
them along with a fixed set of components.

Enter custom command buffers!

These command buffers focus on specific operations, and offer improved playback
performance as a result. `EnableCommandBuffer` and `DisableCommandBuffer` can
optimally enable and disable entities and their `LinkedEntityGroup` buffers.
`DestroyCommandBuffer` can destroy entities en masse. `InstantiateCommandBuffer`
can instantiate entities and initialize several components in one go. And
`AddComponentsCommandBuffer` is especially well-suited for adding cleanup
components, since it can detect destroyed entities and create temporary entities
for cleanup systems to process.

All of these command buffers can be played back by the `SyncPointPlaybackSystem`
(which can play back `EntityCommandBuffers` too). You can fetch this using
`LatiosWorldUnmanaged.SyncPoint.` And you don’t even have to invoke
`AddJobHandleForProducer()` or touch a singleton. Dependency management is fully
automatic. All that boilerplate is gone. As the title says, this sync point is
smart!

You can also play back these buffers manually.

See more: [Custom Command Buffers and
SyncPointPlaybackSystem](Custom%20Command%20Buffers%20and%20SyncPointPlaybackSystem.md)

### Rng and RngToolkit

There are three common strategies for using random numbers in DOTS ECS. The
first is to store the `Random` instance in a singleton, which prevents
multithreading. The second is to store several `Random` instances in an array
and access them using `[NativeThreadIndex]` which breaks determinism. The third
is to store a `Random` instance on every entity which requires an intelligent
seeding strategy and consumes memory bandwidth.

There’s a way better way!

`Rng` is a new type which provides deterministic, parallel, low bandwidth random
numbers to your jobs. Simply call `Shuffle()` before passing it into a job, then
access a unique sequence of random numbers using `GetSequence()` and passing in
a unique integer (`chunkIndex`, `entityInQueryIndex`, ect). The returned
sequence object can be used just like `Random` for the remainder of the job. You
don’t even need to assign the state back to anything.

`Rng` is based on the Noise-Based RNG presented in [this GDC
Talk](https://www.youtube.com/watch?v=LWFzPP8ZbdU) but updated to a more
recently shared version:
[SquirrelNoise5](https://twitter.com/SquirrelTweets/status/1421251894274625536)

However, if you would like to use your own random number generation algorithm,
you can use the `RngToolkit` to help convert your `uint` outputs into more
desirable forms.

See more: [Rng and RngToolkit](Rng%20and%20RngToolkit.md)

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

The design outlines a pattern you can follow that allows you to extend the Smart
Blobber mechanisms to your own blob types. The process is thoroughly documented
and all APIs contain detailed XML documentation. You can obtain fully parallel
and Burst-compiled blob asset generation without having to worry about Unity’s
`BlobAssetStore` caching and deduplication.

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

### DynamicHashMap

This is a wrapper around a `DynamicBuffer` that provides hashmap-like
functionality. Unlike other implementations, this implementation can correctly
handle serialization of Entity and blob asset references.

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
methods. You can instead pass around the `ComponentBroker` by ref, keeping your
method arguments clean. Meanwhile, `TempQuery` allows you to evaluate an entity
query on-the-spot in the job and iterate over chunks and entities. It isn’t as
performant as cached entity queries created on the main thread, but it can still
be handy.

These features are especially useful for mods or scripting solutions such as
Unika.

See more: [Component Broker](Component%20Broker.md) and [Temp
Queries](Temp%20Queries.md)

### Extensions, Exposed, and ECS Cleanups

Sometimes Unity is missing API for no good reason other than DOTS still being
under development. And sometimes, I need this missing API. Sometimes this can be
fixed with extension methods. Sometimes this requires extending the package
directly using asmrefs. The former can be found in the Utilities folder, and the
latter shows up in the `Unity.Entities.Exposed` namespace.

There are also situations where Unity’s package just does really dumb things
that you probably weren’t aware of. For example, all prefabs will allocate on
the heap when instantiated because of an included `LinkedEntityGroup`, even if
they have no children. Or baking systems that use `WorldUpdateAllocator` will
continuously allocate until the baking world is destroyed.

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

## Near-Term Roadmap

-   `ThreadStackAllocator` `AllocatorManager` support
-   Blob prolonged retention system
-   Entity create/change/destroy pipeline
-   Subscene loading world bootstraps
-   Fixed rate time management
-   Bootstrap Profiles
    -   Allow multiple bootstraps per project for samples and tests
-   More custom command buffer types
-   Improved collection components
    -   Default initialization interface
    -   Get as ref
    -   Inspectors
-   Profiling tools
    -   Port and cleanup from Lsss
-   Reflection-free improvements
-   Job-friendly safe blob management
