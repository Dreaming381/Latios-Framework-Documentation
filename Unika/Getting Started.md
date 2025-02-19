# Getting Started with Unika

Unika’s scripting solution may be intimidating if you simply look at the source
code. However, there are only a few small snippets you need to remember to be
effective. This guide will present you with all the basics you care about, and
then leave it up to you to experiment.

## Pre-Defined Interfaces

Unika defines a lot of interfaces. Most of these interfaces are used to define
extension methods that round out Unika’s API and reduce boilerplate in your
code. With that said, it is easy to get lost when searching for interfaces and
types. There are only 7 interfaces your types ever need to implement:

-   IUnikaScript
-   IUnikaInterface
-   IScriptFilter
-   IScriptResolver
-   ICachedScriptResolver
-   ScriptTypeExtraction.IReceiver
-   ScriptTypeExtraction.ITypeReceiver

Of these, all but the first 2 are for niche use cases. You can mostly ignore
them. Any other interface you encounter will either be defined by Unika types or
automatically added to your types via source generators. Such interfaces are
used by extension methods.

## Creating Your First Script

If you haven’t done so already, in the Assets menu, create a new Latios
Bootstrap appropriate for your project. The bootstrap will install some systems
to handle entity serialization for us. These systems aren’t always right for
every project, but they will be sufficient for our purposes.

Now that you’ve done that, in the Assets menu, create a Unika Script and name it
`SayScript`. The template will generate two types named `SayScript`, but placed
in two separate namespaces named `Authoring` and `Scripts` respectively. Both
types are declared `partial`, because both are processed by source generators.
These technically do not need to be in the same file, and defining the authoring
type isn’t a hard requirement.

In the editor, create a subscene. Then add a Game Object to it. Attach your
newly-defined `SayScript` to the Game Object. When you do this, you should see a
*Script Buffer (Unika)* was added as well. That is what will allow the entity to
hold scripts.

Let’s add a method to our script. Add this to the `SayScript` in the `Scripts`
namespace.

```csharp
public void Say()
{
    UnityEngine.Debug.Log("Hello. This is Unika speaking.");
}
```

## Running Your Script

Next, let’s create a system that schedules an `IJobEntity` job. The `Execute()`
method should ask for `Entity` and a `ref DynamicBuffer<UnikaScripts>`.

Next, we want to iterate over all scripts in the buffer. However, it is
currently in the wrong form. To get it into the correct form, we need to call
`AllScripts()` on it and pass in our entity. This creates an
`EntityScriptCollection`, which is the main way to navigate and index scripts on
an entity.

`EntityScriptCollection` is something we can iterate in a `foreach`. However,
this gives us type-agnostic `Script` instances. If we want our `SayScript`, we
need to cast the `Script` to a `Script<SayScript>`. We can do this via the
extension method `TryCastScript<SayScript>()` which returns a `bool` if our cast
was successful and provides the result in an `out` parameter.

This is boilerplate-heavy, and is also not the most performant. Instead, we can
ask `EntityScriptCollection` to provide a filtered list of scripts that are of
our requested type. From the `EntityScriptCollection`, we can call
`OfType<SayScript>()`. This is also something we can iterate in a foreach, but
now we get `Script<SayScript>` results.

`Script<SayScript>.valueRW` provides a reference to the actual `SayScript`
instance. We can now call our `Say()` method. Here’s our full job:

```csharp
[BurstCompile]
partial struct UnikaJob : IJobEntity
{
    public void Execute(Entity entity, ref DynamicBuffer<UnikaScripts> scriptsBuffer)
    {
        foreach (var sayScript in scriptsBuffer.AllScripts(entity).OfType<SayScript>())
        {
            sayScript.valueRW.Say();
        }
    }
}
```

You can schedule this job either single-threaded, or in parallel. It should work
just fine. Try it out by entering play mode!

The final code is very simple, but there were a lot of details we covered. The
key points are that we get an `EntityScriptCollection` from calling
`AllScripts()`, and this gives us access to all the scripts on the entity. These
come in the form of a generic handle called `Script`, and for instances that are
`SayScript`, we need to convert the handle into a `Script<SayScript>` so that we
can access the actual instance. We can cast and check, or we can use an
`OfType<SayScript>()` filter. Spend some time here to play around with the code
and ensure you understand these concepts, because we will continue to build on
them.

## Referencing Another Script

Let’s create another script named `RespondScript`. This time, we will give it a
method `Respond()` like this:

```csharp
public partial struct RespondScript : IUnikaScript
{
    public void Respond()
    {
        UnityEngine.Debug.Log("Hi Unika. Nice to meet you.");
    }
}
```

Add this script to the same Game Object we added `SayScript` to.

Now, we want `SayScript` to reference a `RespondScript` and call `Respond()`. To
reference another script, we can use a `ScriptRef`. This references any script.
To specifically specify we want a `RespondScript`, we use
`ScriptRef<RespondScript>`.

`Script<T>` is kinda like `RefRW<T>`, in that it can be invalidated by
structural changes or job dependencies. That’s why we use
`ScriptRef<RespondScript>`, which is serializable. But we must convert our
`ScriptRef<RespondScript>` into a `Script<RespondScript>` when we want to call
`Respond()`. This process of converting a `ScriptRef` into a `Script` is called
*resolving*.

There are two ways to resolve a `ScriptRef`. If we know the entity the
`ScriptRef` belongs to (a property of `ScriptRef` we can read), then we can use
the `EntityScriptCollection` of that entity to resolve the reference and get a
`Script`. The other option is to use a dedicated resolver, such as a
`LookupScriptResolver`. We pass either of these into the `Resolve()` or
`TryResolve()` methods of our `ScriptRef`. The former `Resolve()` method will
throw an exception if it fails.

In our case, we know the script belongs to the same entity, so we will pass in
the `EntityScriptCollection`. But actually, we can instead pass in the
`Script<SayScript>`. This has the property `allScripts` which is the same
`EntityScriptCollection`. The reason to prefer `Script<SayScript>` is that it
also contains the script’s metadata, which we’ll cover later.

Here’s what `SayScript` looks like now:

```csharp
public partial struct SayScript : IUnikaScript
{
    public ScriptRef<RespondScript> respondScript;

    public void Say(Script<SayScript> me)
    {
        UnityEngine.Debug.Log("Hello. This is Unika speaking.");
        respondScript.Resolve(me.allScripts).valueRW.Respond();
    }
}
```

Back in `Authoring`, we can simply define a reference to a `RespondScript` as a
public or serialized field. Because we are in authoring, this will reference the
authoring `MonoBehaviour` for `RespondScript`. When we create the
`Scripts.SayScript` in `Bake()`, we can simply call `GetScriptRef()` on our
authoring `RespondScript` and pass in the `IBaker`. Note that we do not need to
call `DependsOn()` here, as `GetScriptRef()` will register all required baking
dependencies for us.

Because we used the `EntityScriptCollection` to resolve our `ScriptRef` on the
same entity, we can safely schedule our job in parallel. If we had used a
`LookupScriptResolver`, we’d get safety errors. However, a
`LookupScriptResolver` in a single-threaded job would allow us to reference a
`RespondScript` on a different entity in our scene. Feel free to give that a
try!

### Resolving ScriptRef Mutates

`ScriptRef` contains a cache about the whereabouts of the script it references
inside the buffer. If scripts were added to or removed from the target entity at
runtime, this cache can get out of date, and Unika will have to do a slower
search to find the script. When this slower search happens, the `ScriptRef`
cache will be updated to the new location.

This can sometimes be problematic, especially if the `ScriptRef` comes from an
`in` parameter or some other `readonly` context. To get around this limitation,
copy the `ScriptRef` to a local variable, then resolve the local version. This
won’t update the original cache, but it will circumvent the compiler errors.

Caches remain valid through serialization and instantiation. If you do not add
or remove scripts at runtime, then the cache will always have the right
location. That means even your first resolve of a `ScriptRef` may still have an
up-to-date cache.

### Assignments and Comparisons

So far, we have shown conversion from `Script` to `Script<T>` and from
`ScriptRef<T>` to `Script<T>`. But what about going the other direction?

`Script<T>` can be assigned to any `Script` or `ScriptRef<T>`. And any of those
three can be assigned to a `ScriptRef`.

All these types support various equality tests against each other and can all be
used in hashmaps. Additionally, they all define a `Null` instance and all expose
an `entity` property.

## Time to Polymorph

So far, everything we did had a hard requirement of knowing the exact script
type to work with it. That’s far from flexible, and doesn’t really add much over
what we can do with vanilla ECS.

But what if instead of searching for scripts of a specific type, we searched for
scripts that implemented that interface? And then what if we could call those
interface methods without ever knowing what type the script actually was?

We’re going to do just that.

Let’s define the following interface:

```csharp
partial interface ISay : IUnikaInterface
{
    void Say(ISay.Interface self);
}
```

Alright. There’s already a lot to unpack here. The first thing to note is that
we have a `partial interface`. That means this interface is being expanded upon
by source generators. That’s also why this interface inherits the
`IUnikaInterface`.

There’s also `ISay.Interface` being passed as a parameter in the `Say()`
definition. Interface is a type created by the source generator. Think of it
like the interface equivalent to `Script<T>` in that `Interface` is a handle to
a script that implements the interface. As you might expect, the source
generator also created `ISay.InterfaceRef`.

Let’s modify `SayScript` to implement `ISay`. `ISay.Interface` also has the
`allScripts` property, so we don’t need to modify the body of `Say()` at all.

In our job, we want to filter our scripts to the ones implementing `ISay`. We do
that using `Of<ISay.Interface>()`. This is a really important point to keep in
mind. When casting or filtering for an interface, we always use the generated
Interface as the generic argument.

Now we have an `ISay.Interface` instance in our foreach. And from it, we want to
call our `Say()` method. `ISay.Interface` implements `ISay`, so we can just do
that.

```csharp
foreach (var sayInterface in scriptsBuffer.AllScripts(entity).Of<ISay.Interface>())
{
    sayInterface.Say(sayInterface);
}
```

Our job is no longer referencing any explicit script types from the `Scripts`
namespace. That means we can add all sorts of new scripts that implement `ISay`,
each doing their own thing. And our job won’t have to be updated at all to work
with them.

### IUnikaInterface Capabilities and Limitations

At this point, you might have a lot of questions about what you are allowed to
do. You can do a lot.

`IUnikaInterface` supports multiple methods, properties, and indexers. For
methods and indexers, you can pass up to 7 parameters (let me know if you need
more). It supports `in`, `ref`, and `out` parameters as well as `ref` and `ref
readonly` return values. However, `ref struct` parameters or return values are
**NOT** supported.

`IUnikaInterface` is allowed to inherit other interfaces, and all of those
interface methods will also be supported polymorphically. Scripts are also
allowed to implement multiple different interfaces of type `IUnikaInterface`.

Converting a `Script<T>` to an `Interface` is not implicit, but there should be
an extension method generated called `ToInterface()` which produces a temporary
type assignable to any `Interface` the script implements.

In authoring, you can get an `InterfaceRef` by any authoring component that
implements `IUnikaInterfaceAuthoring<T>` where `T` is the `InterfaceRef` type
you care about. The implementation for this is explicit, so you will need to
cast the authoring component to a `IUnikaInterfaceAuthoring` interface instance
to call that method.

## Script Metadata

Let’s imagine we have an interface named `IOnCollision`, and we have an entity
we need to invoke that interface on for all scripts implementing that interface.
However, we have 30 scripts on that entity, and only two of them implement that
interface. In addition, some of the scripts have a lot of data stored in them.
Traversing the entire scripts buffer to find the 2 scripts we care about would
be extremely wasteful of memory bandwidth. How does Unika deal with this?

The answer is that Unika divides up the scripts buffer into three separate
regions:

-   Master Header
-   Script Headers
-   Script Instance Data

The Master Header contains special bookkeeping data such as the number of
scripts as well as a filter mask for quickly ruling out an entire buffer for
script and interface searches.

Script Headers contain the metadata such as what type each script is, and where
its instance data is located. They also contain a filter mask for quickly ruling
out implemented interfaces. Thus, when searching for scripts that implement
`IOnCollision`, Unika reads the Master Header, and then searches the Script
Headers. This greatly reduces the amount of memory Unika must tread through
during its search.

Now let’s assume that we instead have an `IUpdate` interface, and we wanted the
concept of scripts being “enabled”. Perhaps lots of scripts implement the
`IUpdate` interface, but only a small number are enabled at a time. Making a
virtual call to check if each script is enabled would be very expensive. We can
do better.

Each Script Header stores several user values which you can use for whatever
purpose you want. They are named `userFlagA`, `userFlagB`, and `userByte`. The
first two are `bool` values (bit-packed in the header) while the last is a full
`byte` which could be used to store enumerations or counters. These values are
provided to you to read or modify by any `Script`, `Script<T>`, or `Interface`.
You may have also seen in the `Authoring` `MonoBehaviour` where these values can
be initialized during baking.

You can incorporate these values into your foreach statement when searching for
script types or interfaces. You do this using the `Where()` method. You can
chain up to 8 `Where()` methods at a time. Each `Where()` method requires an
`IScriptFilter`. You can find a prebuilt collection of filters in the static
`ScriptFilter` class.

If we treated `userFlagA` as the enabled state, we could write our search like
this:

```csharp
foreach (var needsUpdate in scriptsBuffer.AllScripts(entity).Of<IUpdate.Interface>().Where(ScriptFilter.UserFlagATrue))
```

**Warning:** If you create your own custom `IScriptFilter`, Unika will sometimes
provide it false positives for whatever initial type or interface it is
searching for. Filters run before the final matching step for performance
reasons.

## Entity Serialization

By default, Unika handles serialization of most types at baking time and
deserialization within a subscene `ProcessAfterLoad` system. However, because
entities can be remapped during instantiation, special care is needed to ensure
this remapping happens correctly.

The default systems we enabled in the bootstrap assume that all entities are
instantiated or loaded during `InitializationSystemGroup`. Therefore, one system
at the beginning of `InitializationSystemGroup` is responsible for serializing
any entity with the `UnikaEntitySerializationController` component enabled (it
is an `IEnableableComponent`). You must enable this component for any entity you
wish to instantiate if any of the script’s references to entities (including
`ScriptRef` types) may have changed since the last serialization or load. It
will be disabled again by the deserialization system which updates in
`PostSyncPointGroup`.

If you only ever instantiate entities with scripts from prefabs, you may not
need to touch serialization.

If the built-in entity serialization systems do not match your needs, remove
Unika from the bootstraps and look at the `ScriptSerialization` static class to
drive entity serialization. You might also be interested in that class if you
need to perform saving or network synchronization.

## Unika in Projects

Now that we’ve covered the various essentials to Unika, you might be wondering
how all this might fit into the grander ECS spectrum. I’ll provide some hints
and ideas here, but it will be up to you to decide if these are good ideas for
your project.

### Data Access

An important detail to keep in mind is that unlike `MonoBehaviours` which in
some way or another can access just about anything, Unika scripts are limited by
what is provided through the interfaces. If you never give scripts access to any
ECS data, scripts will never be able to modify ECS components. If you only ever
give access to a handful of component types, then scripts can only interact with
those component types. Passing in an `EntityManager` or an `EntityCommandBuffer`
will give scripts free reign over whatever entities they can find. What is
exposed and what is hidden from scripts should be an intentional choice based on
your project and team.

From a practical standpoint, many projects may wish to create context structs
which contain a [Component Broker](../Core/Component%20Broker.md), game time
values, various key entity references, perhaps a command buffer, and maybe even
the archetypes array for [Temp Query](../Core/Temp%20Queries.md).

### Sparse Logic Updates

One area Unika is very good at is running one-off logic segments efficiently.
Rather than scheduling a job for each piece of logic, Unika allows many
different pieces of logic to all execute within the same job. It is often a good
idea to run a single-threaded Unika job alongside a heavy computation job such
as pathfinding.

Good candidates for sparse logic can be things involving player handling,
especially if the player character is composed of a hierarchy. Camera, UI, game
rules, boss logic, and global event managers may all be good candidates.

Unika is also good at events that happens sparsely, especially if such events
may need to be handled one of many ways. An example of this is an interaction
system in an RPG. While an ECS system may perform the logic of detecting that an
interactable should be interacted with, it may be more efficient to let Unika
handle the interaction rather than schedule a job for each possible handling
logic.

### Modular Sequencing

With polymorphism, Unika can be quite effective at handling modular sequenced
logic, such as tween sequences, state machines, behavior trees, and instruction
queues. While such problems can usually be solved in ECS, making them efficient
can prove challenging. Most of these problems don’t often need “absolute
performance”. A modestly-performing solution off the main thread is often good
enough and can save months of development time and frustration. With jobs and
Burst, Unika provides significantly better performance than traditional Unity
OOP.

Don’t forget that `ScriptRef` and `InterfaceRef` can be stored in ECS
components, buffers, and other collections.

### Perfect Moment Execution

There are times when it can be much more efficient to process things “in the
moment” rather than defer to a later point. A good example of this is collision
and contact handling events. In Unity Physics, collisions and contacts need to
be dumped into buffers, and then scanned by subsequent jobs for any logic that
needs to modify it. Much of the saved data may actually be for pair combinations
the special logic doesn’t actually care about. This is very hard on memory, and
is somewhat painful to make efficient. With Unika, scripts can evaluate
collision and contacts right when they are computed and fresh on the stack. No
more scanning, and no more just-in-case memory usage.

### Scripting

Well, yeah. The very first sentence of this guide described Unika as a
“scripting solution”. But what does that actually mean?

It generally means that once you have a few ECS systems to drive Unika, scripts
can be much faster to prototype and iterate with. And they are also friendlier
to tech artists or those who struggle with DOTS.

## Advanced APIs

Unika contains a few advanced APIs when you need a little more power. They are
discussed briefly here to make you aware of them. But make sure to read the XML
docs to understand how to use them.

### Multi-Script Baking

`UnikaMultiScriptAuthoring` is an abstract class which you can inherit instead
of using the authoring component from the template. This authoring component
lets you set up multiple scripts at once dynamically that may reference each
other. You might choose to use this API when baking complex configurations like
state machines or graphs from a `ScriptableObject`, and the number and types of
scripts are somewhat dynamic.

There are also extra authoring extension APIs. `IBaker.AddScript()` can add a
script to any `UnikaScriptBufferAuthoring` including on different `GameObjects`
from what is currently being baked.

`EntityManager.GetBakedScriptInPostProcess()` allows you to patch scripts that
needed blobs computed via smart blobbers before they get merged into the script
buffer.

And `IBaker.CreateScriptBuffersForAdditionalEntity()` lets you set up scripts
for an entity created entirely by the baker. In this case, you get the
`DynamicBuffer<UnikaScripts>` directly and can use the runtime APIs to populate
it.

### Runtime Adding and Removing of Scripts

The static `ScriptStructuralChange` class allows for adding and removing a
script to the `DynamicBuffer<UnikaScripts>` at runtime. Be very careful though,
because if you call this from inside a script on the same buffer the script
lives in, you can corrupt your memory and even crash Unity.

### Script Casting Generically

Many of the extension methods for casting and converting various script handles
can be found in the static `ScriptCast` class. You can also use these methods
directly in your own generic code if the need arises.

### Script Reflection

The static class `ScriptTypeExtraction` provides various managed (not
Burst-compatible) utilities for converting a `Script` or a `System.Type` to a
concrete generic that can be forwarded back to your own logic. You might use
this for custom editor tooling or debugging where you want the specific script
type details for every instance in a scripts buffer.
