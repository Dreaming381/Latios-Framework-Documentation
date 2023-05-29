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

### EntityDataCopyKit

Ever want to copy a component from one Entity to another using its
`ComponentType`, perhaps obtained by comparing two entities’ archetypes? I did,
so I wrote this. It uses asmref internal access hacks to implement the features
not possible by the public API (Shared Components).

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
containing “viable” options, well you can do those things here.

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
`worldGlobalEntity`. All subscenes loaded are forced to load synchronously,
ensuring that settings entities are present right away.

If you want an entity to stick around (besides the `worldBlackboardEntity` which
always sticks around), you can add the `DontDestroyOnSceneChangeTag`.

Now you can build your Mario Party clone, your multi-track racer, or your fancy
5-scene credits system in ECS with the ease you had in `MonoBehaviour`s. Oh, and
having most scenes be almost exclusively `MonoBehaviour`s while a couple of
scenes use ECS is totally a thing you can do, especially since you can check
`CurrentScene` inside `ShouldUpdateSystem`. For all you out there trying to
shoehorn ECS into your Mono game, shoehorn no more!

*This feature is no longer enabled by default for any bootstrap except the
DreamingBootstrap. It is also not compatible with NetCode and may require tweaks
to work correctly with systems that rely on singletons, such as Unity Physics.*

See more: [Scene Management](Scene%20Management.md)

### Collection Components

Why in the world does Unity think that collections only belong on systems or
singletons? Did they not watch the Overwatch ECS talks?

All joking aside, they support them in three different ways:

-   Class components implementing `IDisposable`, which works but allocates GC
-   Collections which you have to be extra careful using to avoid memory leaks
    and don’t provide automatic dependency management
-   Singletons which only provide dependency management for contained
    collections (which is really bizarre)

I wasn’t really satisfied with these solutions, so I made my own. They are
structs that implement `ICollectionComponent`.

**Warning: Managed components and collection components are not real components.
The real component is the ExistComponent nested type defined by source
generators.**

You can query for the ExistComponent in your own code to find entities with a
collection component. You can then access the collection component using the
`LatiosWorldUnmanaged` methods.

Collection components have this nice feature of automatically updating their
dependencies if you use them in a system.

See more: [Collection and Managed Struct
Components](Collection%20and%20Managed%20Struct%20Components.md)

### Managed Struct Components

So you got some SOs or some Meshes and Materials or something that you want to
live on individual entities, but you don’t want to use Shared Components and
chop your memory and performance into gravel. You also don’t want to use class
`IComponentData` because that’s GC allocations every time you instantiate a new
entity and you know how to reference data on other entities using Entity fields.
Really, you just want GC-free structs that can store shared references. You
can’t use them in jobs, but that’s not the concern.

Meet `IManagedStructComponent`. It is a struct that can hold references. You can
get and set them using `LatiosWorldUnmanaged.Get/SetManagedStructComponent` and
friends. They work essentially the same as `ICollectionComponent` except without
the automatic dependency management because there’s no dependencies to manage.

See more: [Collection and Managed Struct
Components](Collection%20and%20Managed%20Struct%20Components.md)

### Math

What framework would be complete without some math helpers? Not this one.
Overly-used algorithms and some SIMD stuff are here. Help yourself!

See more: [Math](Math.md)

### Extensions and Exposed

Sometimes Unity is missing API for no good reason other than DOTS still being
under development. And sometimes, I need this missing API. Sometimes this can be
fixed using an extension method. Sometimes this requires extending the package
directly using asmrefs. The former can be found in the Utilities folder, and the
latter shows up in the `Unity.Entities.Exposed` namespace.

### Fluent Queries

Fluent syntax for expressing EntityQueries was a big improvement. However, every
iteration so far has lacked a way to extend it. This implementation not only
provides clean Fluent syntax for building EntityQueries, but it is also
extensible so that library authors can write patch methods for their
dependencies.

Fluent Queries have a concept of “weak” requests, or requests that can be
overridden by other expressions. For example, an extension method may request a
weak readonly `Translation`. If a `Translation` request already exists (readonly
or readwrite), the weak request will be ignored.

There are similar mechanisms for handling “Any” requests and “Exclude” requests.

You begin a Fluent chain using the `Fluent` property on `SubSystem` and
`SuperSystem` or by invoking `Fluent()` on an `EntityManager` or `SystemState`.
To get the resulting `EntityQuery`, call `Build()`.

See more: [Fluent Queries](Fluent%20Queries.md)

### Smart Sync Point and Custom Command Buffers

`EntityCommandBuffer` is a powerful tool, but it has some limitations.

First, it has no equivalent for `EntityManager.SetEnabled()` in parallel jobs.
While this can be replicated by attaching or detaching the Disabled component
directly, one would also have to manage the `LinkedEntityGroup`, which could
change between command recording and playback.

Enter `EnableCommandBuffer` and `DisableCommandBuffer`. They are quite limited
in that they can only handle one type of command each, but they do it right!

The second issue comes when instantiating new entities. Often times, the entity
does not just need to be instantiated, but also have some of its components
initialized. This is done one-by-one in the `EntityCommandBuffer` which can be
slow.

Enter `InstantiateCommandBuffer`. You can use this command buffer to instantiate
entities and initialize up to 5 components. You can also add an additional 15
components on top. It uses batch processing for increased speed.

Lastly, there’s a `DestroyCommandBuffer`. This command buffer may provide a
speedup in some circumstances.

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

So instead, use `EntityWith<T>` and `EntityWithBuffer<T>` instead! They work
just like normal `Entity` references, except you can gather additional context
about them. An `EntityWith<Prefab>` should probably be instantiated. An
`EntityWith<Disabled>` needs to be enabled at the right moment. An
`EntityWith<LocalToWorld>` is a transform to spawn things at or attach things
to.

### Smart Blobbers

Blob Asset Baking is tough to get right, especially if you want to cache
expensive computation and generate blobs in parallel via baking systems. And
what if multiple bakers want to leverage this caching and parallel baking, but
do something custom with the results?

Smart Blobbers solve this problem and make blob asset baking simpler, especially
for large projects with complex baking dependencies. They provide a built-in
mechanism for Bakers to request blobs to be built. Those requests can later be
resolved into real `BlobAssetReference<>` values. Smart Bakers provide a
mechanism to propagate bake context beyond a Baker’s scope so that blob assets
can be resolved correctly without having to write custom baking systems.

The design outlines a pattern you can follow that allows you to extend the Smart
Blobber mechanisms to your own blob types. The process is thoroughly documented
and all APIs contain detailed XML documentation. You can obtain fully parallel
and Bursted blob asset generation without having to worry about Unity’s
convoluted blob asset conversion APIs.

See more: [Smart Blobbers](Smart%20Blobbers.md)

### UnsafeParallelBlockList

This container is really unsafe. It was originally only meant for internal
purposes. But here it is on this page. Why?

It is fast! Really fast!

So fast that people were finding ways to use it anyways, whether that be copying
the code or modifying the package. Now it is public API so people don’t have to
do those workarounds anymore.

Just be warned. It really truly is a gun eager to put a bullet through your
foot.

## Known Issues

-   `IManagedComponent` and `ICollectionComponent` are not true components. They
    instead rely on a source-generated `ExistComponent` to signify presence with
    a specific entity.
-   Compile errors are generated when using .Net Framework. Use .Net Standard.
-   IL2CPP requires the IL2CPP Code Generation to use the “Faster (smaller)
    builds” option.
-   Blackboard Entities do not retain blob asset reference counts.
-   Blackboard Entities and `EntityDataCopyKit` do not handle enable bits
    correctly.

## Near-Term Roadmap

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
