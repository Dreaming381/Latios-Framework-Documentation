# QVVS Transforms V2 Design Discussion

QVVS Transforms is getting a redesign. This document discusses why a new design
is needed, what the new design will be, and what challenges remain.

## A Recap on the Current Design

Currently in QVVS, a hierarchy is composed of entities, which specify their
`Parent`. Such entities also may have a `LocalTransform`. All entities belonging
to a hierarchy have a `WorldTransform`. In this state, such entities are not
ready to have their transform modified. They first need to be processed by the
`TransformSuperSystem`. The `TransformSuperSystem` is responsible for adding the
`PreviousParent` and `ParentToWorldTransform` components.

The `PreviousParent` component is how subsequent updates can detect that the
parent changed. This is used to update the `Child` buffer of parents. If you
assign a new `Parent`, the `Child` buffer of the previous parent and new parent
is stale until the next `TransformSuperSystem` update. This means it is never
possible to truly know what all the descendants are of any entity without
querying all entities with parents, and walking up the ancestors of each one
until you either determined they are a descendant, or you encounter the root.
This is not performant. So instead, developers must structure the order of their
logic so that modifying parents happens before `TransformSuperSystem`, and
reading the hierarchy happens afterwards.

`ParentToWorldTransform` allows `TransformAspect` to guess at what should happen
to `LocalTransform` when modifying the world-space transform, and similarly to
the `WorldTransform` when modifying the local-space transform. These are never
perfect calculations, because an ancestor transform change will not be accounted
for. However, it can at least be guaranteed that for each property of the
transform (position, rotation, scale, ect) that either the `LocalTransform` or
`WorldTransform` are correct at all times. You can choose which using the
`HierarchyUpdateMode` component. The `TransformSuperSystem` is responsible for
updating the incorrect values and updating the `ParentToWorldTransform` to
reflect the latest state of parent. This means that the transform hierarchy is
only truly correct after a `TransformSuperSystem` update up until the first
modification of any non-leaf transform or reparenting. After that, you are
technically working with stale data.

`TransformSuperSystem` is a global operation, updating all entities of all
hierarchies at once, and using a sync point to process reparenting changes. The
reason it is global is because at the start of the update, it has no idea which
entities belong to which hierarchy. The caches may be stale. This means that
every entity needs to be visited, even if only a small subset of roots were
modified. Extreme Transforms mode heavily optimizes this algorithm at scale. But
it still does not optimize small incremental changes.

Lastly, there a motion history components that are strategically updated at a
specific point in the frame to copy `WorldTransform` to `PreviousTransform`, and
copy `PreviousTransform` to `TwoAgoTransform`. Right after this point, it is
expected the user will want to modify the `WorldTransform`. This rotation
happens right at the beginning of `SimulationSystemGroup`.

### Current QVVS Transforms Strengths

This current solution, despite its flaws, is great for singleplayer experiences.

A typical game loop will spawn entities, then run `TransformSuperSystem` to set
the spawned transforms into an initial state. After that, motion history
updates. Then, various systems can update the motions of entities using
`TransformAspect`. The `PreviousTransform` is available to help bypass logical
ordering issues. And then, the `TransformSuperSystem` updates a second time to
integrate all the motion. Lastly, with up-to-date transforms, various spatial
queries and events can read the updated transforms to advance internal state.
Rendering can occur after either `TransformSuperSystem` update, depending on
project needs.

In this style of game loop, `TransformSuperSystem` only has to run twice in a
frame, using its optimized global algorithm. That’s usually acceptable
performance. In addition, organizing systems around the `TransformSuperSystem`
is often fairly straightforward. Reading and writing transforms is always fast
and easy because of `TransformAspect`. And if something doesn’t quite fit right,
the approximation `ParentToWorldTransform` is often “close enough”.

In the end, the current solution is easy to use with `TransformAspect`, very
fast for reading and writing transforms inside `IJobEntity`, easy to organize
systems, and in many cases just works.

## Why the Current Design Will Fail

While the use cases supported today are great fits for the current design of
QVVS Transforms, the future of ECS and the Latios Framework shape a more hostile
future for the current design.

### No More IAspect

Unity desires to remove `IAspect`, without any replacement. While in theory this
is not a bad move, in practice, it exposes various other failures of the
Entities package API that are likely to remain unaddressed. A big one in
particular is that `IAspect` is currently the only way to specify optional
components in an `IJobEntity`. `TransformAspect` alters its behavior based on
the presence of optional components. Unfortunately, it is extremely bugprone to
trust users to correctly specify these optional components when they are
present. Thus, ease-of-use is no longer an advantage of the current design.

### Awkward Dynamic Environment Traversal

In some types of games, it is common to animate environments, and then have
physics and the character controllers react to these movements. However,
animations within a hierarchy animate the local space, which means the hierarchy
needs to be updated for physics and the character controllers to see the correct
collider positions of the environment. Camera systems can also suffer similar
issues, wanting to react based on the updated transforms of everything else. All
of this means more `TransformSuperSystem` updates are required each frame, which
hurts performance.

### Incremental and Static Update Issues

`TransformSuperSystem` doesn’t know which entities have stale transforms. It
doesn’t know which entity is the most ancestral of a dirty subtree within any
given hierarchy. It doesn’t even know if the hierarchies are even correct. And
because of this, it needs to explore the entire hierarchy every time it updates.
Even if all that changed were a few new entities being instantiated, everything
needs to be revisited.

Where this becomes especially problematic is in a networking context where only
a small subset of entities are rolled back to a previous state, and then
simulated for multiple timesteps in a single frame, often based on the enabled
state of the `Simulate` component. To even do this correctly, the `Simulate`
flag needs to be propagated throughout the whole hierarchy. Since this will
often be a top-down operation, a system responsible for performing this
propagation would need an up-to-date `Child` buffer. But once again,
`TransformSuperSystem` doesn’t know the most ancestral entity of any dirty
subtree. It would have to visit every entity to do this propagation. And in a
networking prediction loop that might update 30 timesteps per frame, this would
add up fast!

### The Familiar Transform Model Is Changing

The QVVS Transforms model borrows from the Unity Transforms model in several
ways, which helps make the model familiar. The idea of parent changes being
deferred to the next `TransformSuperSystem` update is a Unity Transforms thing.
The idea of explicit hierarchy synchronization also comes from Unity Transforms.
And the idea of separate local and world-space transform components, as well as
the separate `Parent` component is all borrowed from Unity Transforms.

But eventually, Unity will move to a new unified transform system between ECS
and Game Objects. When that happens, the model is going to completely change.
Transforms will be an engine-concept interacted with indirectly through via
handles. And the current QVVS Transforms model may start to feel outdated and
slowly become unfamiliar to developers.

### In Short

While none of these issues on their own warrant a redesign, it is when you stack
them all together that things don’t look great. So the question is, would a
different design achieve more than what this design does?

## Design Thought Process for V2

Let’s start with a design that addresses some of the major weaknesses of V1, and
then try to recover the strengths of V1.

### Addressing V1 Problems

First, we want a solution that allows hierarchies to update independently. This
means that every entity will need some way of knowing which hierarchy it belongs
to. It needs some identifier. We also need this identifier to not alias when
instantiating hierarchies. This gives us three options:

1.  Use an integer and cleanup component
2.  Use a hashmap mapping each entity to an integer
3.  Use the root entity as the identifier

(1) requires a reactive system for every entity in a hierarchy. (2) requires
scanning every entity to detect which ones are in the hashmap and which ones
aren’t. (3) can leverage entity remapping on instantiation, so it is only
destruction which requires some kind of reaction.

What is especially interesting about (3) is that it only requires entities react
to destruction if there are entities in the hierarchy not included in the
`LinkedEntityGroup`. That can’t happen until an entity outside the
`LinkedEntityGroup` is explicitly parented, or the hierarchy is prespawned in a
subscene without `LinkedEntityGroupAuthoring` yet still destroyed by gameplay
logic. Reparenting is something explicit that could require going through a
custom API to detect. And prespawned hierarchies are much easier to deal with
because there’s only one point in the frame where subscenes are loaded.

This means that any entity can reference an alive root entity immediately after
instantiation or subscene load, or after some specific transform manipulation
such as reparenting. And if the root is destroyed, either the entity will be
destroyed alongside it (subscene or `LinkedEntityGroup`), or it will have
participated in a reparenting operation, or it will have been detected right
after subscene load reasonably before the root is destroyed again. The
reparenting and subscene load cases provide opportunities to attach a cleanup
component. This means that the root lifecycle is fully tracked by dependent
entities regardless of when and where entities are instantiated and destroyed in
the game loop!

The only caveat is that if an entity in the middle of the hierarchy is destroyed
by itself, then the children won’t immediately detect that they are orphaned and
need to belong to a new hierarchy. A scheduled reactive system can still catch
this, which makes this rare case no worse than (1) and (2). In every other
aspect, (3) is superior. So let’s start with that.

Next, we need to decide where to store hierarchy relationships. We can either
store the parent and children on each entity, or we can store the whole
hierarchy order on the root. The latter has some very obvious advantages. Having
the whole hierarchy in one place means less `DynamicBuffer` allocations, smaller
child archetypes, and a central access for data which is more cache efficient.
And when combined with `EntityQueryMask`, this makes for some very fast queries
to find ancestors or descendants possessing a specific component. And most
importantly, all of the lifecycle guarantees we worked out for the root also
apply for the hierarchy on the root. This means that all of our parent and child
relationships can be self-consistent at all times (other than destroyed entities
hanging around, but these can be caught on-the-fly).

Next, we want to read valid world transforms while potentially modifying other
transforms. Having to keep track of which entity queries had synchronized
hierarchy transforms and which haven’t yet sounds like a recipe for disaster.
What if instead, we enforced a rule where the world transform is always
up-to-date?

Because we know the hierarchy relationships are up-to-date at all times and
accessible from any entity, we can actually do this. There’s some threading
things we’ll need to watch out for, and of course performance will be of
concern. But having this self-consistency at all times will greatly reduce the
cognitive load of developers, compensating for the loss of ease-of-use `IAspect`
provided.

### Designing the Components

Without a doubt, the most important thing to be able to do is read the
world-space transform. Having this component exist on each entity in the
hierarchy seems quite necessary. `WorldTransform` stays in V2.

If a root doesn’t have a hierarchy, then that’s all we need, just like in V1.
However, if it does have a hierarchy, then we need a hierarchy buffer. Each
element `EntityInHierarchy` would contain the Entity, a parent index, the first
child index, the number of children, and maybe some transform inheritance rule
flags. There’s some `EntityInHierarchy` size vs max hierarchy size tradeoffs to
be made yet. This is all the root needs for most cases, which means transforms
only consume 64 bytes of chunk memory per entity, just like V1 again. In the
cases where there’s a risk of the root being destroyed before a descendant, we
can duplicate this buffer with a cleanup version.

Each descendant in the hierarchy will need to know the root entity. And
additionally, it would be helpful to know its element index in the
`EntityInHierarchy` buffer so we don’t have to search for it. This will be the
`RootReference` component.

Descendants also need a local transform…

Actually, no they don’t.

We know that `WorldTransform` is always up-to-date, and we always have a way to
find the parent’s `WorldTransform`. We can accurately calculate the local
transform given only an entity’s `WorldTransform`, and its parent’s
`WorldTransform`.

Not only does this reduce the number of bytes needed per descendant entity in a
chunk, it also means we only need to look up a single `WorldTransform` per
descendant when propagating transforms through the hierarchy. This isn’t quite
as optimal as accessing parents cache-wise, but we partly make up for it with
perfect knowledge of which transforms need to be updated.

Throw in our two motion history components `PreviousTransform` and
`TwoAgoTransform`, and we end up with only 6 component types for transforms
overall.

-   `WorldTransform`
-   `PreviousTransform`
-   `TwoAgoTransform`
-   `EntityInHierarchy`
-   `EntityInHierarchyCleanup`
-   `RootReference`

Unlike V1, where there were some situations where you’d want to modify
components directly, V2 won’t have that. The rule will be you should **never**
modify the transform components directly.

Seeing these components is one thing, but we need to also design the workflows
for interacting with these components in common use cases.

## Functional Use Cases

Let’s work out with each use case what needs to happen, and what the API needs
to be. We’ll start with use cases where we care about functional correctness.

### Reading WorldTransform, PreviousTransform, and TwoAgoTransform

Starting with the easiest, is reading world-space transforms. These already
exist as components that work with all the native ECS APIs for reading. There’s
no change compared to V1 for any of these.

### `Reading Local Transform`

To read a local transform, we need an entity’s `WorldTransform` and
`RootReference` as read-only Then we also need a way to look up the
`EntityInHierarchy` buffer as read-only.

Entities 1.4.2 doesn’t have `IAspect` support, so we’ll need a way to guarantee
that the `WorldTransform` and `RootReference` come from the same entity. The
simplest solution is to require lookups for these as well. We can put all of
this into a static class named `TransformTools`.

```csharp
public static TransformQvs LocalTransformFrom(Entity entity, EntityManager entityManager) => throw new System.NotImplementedException();
public static TransformQvs LocalTransformFrom(Entity entity,
                                              ref ComponentLookup<RootReference>  rootReferenceLookupRO,
                                              ref ComponentLookup<WorldTransform> worldTransformLookupRO,
                                              ref BufferLookup<EntityInHierarchy> entityInHierarchyLookupRO) => throw new System.NotImplementedException();
```

However, there are situations where using lookups is suboptimal. A user may
instead want to promise that the `WorldTransform` and `RootReference` come from
the same entity, because they got it from the Execute parameters of `IJobEntity`
or something. In that case, we can expose another static class
`TransformTools.Unsafe` (it is nested inside `TransformTools`) with the
following methods:

```csharp
public static TransformQvs LocalTransformFrom(Entity entity, in RootReference rootReference, in WorldTransform worldTransform,
                                              EntityManager entityManager) => throw new System.NotImplementedException();
public static TransformQvs LocalTransformFrom(Entity entity,
                                              in RootReference rootReference,
                                              in WorldTransform worldTransform,
                                              ref BufferLookup<EntityInHierarchy> entityInHierarchyLookupRO) => throw new System.NotImplementedException();
```

This will be the pattern for most transform accesses and manipulations. Because
`EntityManager` can replace any lookup access, method overloads that use it will
be omitted from examples here-on out.

There’s still a catch here. What if the parent was destroyed?

We have two options for this. We could start treating the entity as its own
root, and then find some opportunity later to separate it into its own
hierarchy. Or we could reparent it to the next non-destroyed ancestor.

This might seem counter-intuitive, but reparenting is less work. In fact, there
really isn’t any work involved. We just skip over destroyed entities in the
hierarchy. That’s it. There’s no local transforms to update, because those are
implicit. We could potentially rework the `EnityInHierarchy` buffer as an
optimization, but it isn’t required for us to know everything we need to know
about the hierarchy at any time. There won’t be any popping or snapping, because
the `WorldTransform` of the orphaned entity will still be preserved. And this
way, an entity can never escape its hierarchy outside of QVVS V2 APIs.
Reparenting simplifies a lot of the rules.

### Reading Relationships

Since we have to fetch the parent to read the local transform, it should be of
no surprise that reading parent and child relationships from the hierarchy isn’t
much different. Reading the parent is as simple as this:

```csharp
public static Entity ParentFrom(Entity entity,
                                ref ComponentLookup<RootReference>  rootReferenceLookupRO,
                                ref EntityStorageInfoLookup entityStorageInfoLookup,
                                ref BufferLookup<EntityInHierarchy> entityInHierarchyLookupRO) => throw new System.NotImplementedException();
```

This does raise an interesting concern though. What happens if the parent is a
destroyed entity with outstanding cleanup components?

For local transforms, we can detect this by the absence of `WorldTransform`.
However, we won’t know that with `EntityStorageInfoLookup`. However,
`EntityStorageInfoLookup` does provide the `ArchetypeChunk`. That’s technically
enough to call `Has<WorldTransform>()` without the need for the handle, but we
can be slightly more optimal with some asmref tricks to test whether the
underlying `Archetype` of the chunk is a cleanup archetype.

Now, accessing a parent is good and all, but what if then the code wants to
access the parent’s parent? It turns out we don’t just want to return the parent
entity, but also some metadata for future queries. Thus, we define the
`EntityInHierarchyHandle` type. And then we have these methods:

```
public static EntityInHierarchyHandle ParentFrom(Entity entity,
                                                 ref ComponentLookup<RootReference>  rootReferenceLookupRO,
                                                 ref EntityStorageInfoLookup entityStorageInfoLookup,
                                                 ref BufferLookup<EntityInHierarchy> entityInHierarchyLookupRO) => throw new System.NotImplementedException();
public static EntityInHierarchyHandle ParentFrom(EntityInHierarchyHandle entityInHierarchyHandle,
                                                 ref ComponentLookup<RootReference>  rootReferenceLookupRO,
                                                 ref EntityStorageInfoLookup entityStorageInfoLookup) => throw new System.NotImplementedException();
```

`EntityInHierarchyHandle` would then expose an `entity` property.

Obviously, doing repeated accesses in the hierarchy would be constantly looking
up the same hierarchy buffer. We might not need to be so protective to be safe,
because we can always validate that a passed in entity happens to be at the
expected index of the buffer. Consequently, the final API will likely require
some experimentation with ergonomics.

What also requires experimentation is accessing children. The child count is
never truly known, only an upper bound (as some children may be destroyed and
grandchildren may be inherited). How often does the count need to be known vs
just enumerating them on the fly? I’m not sure yet. So this also will be an area
of ergonomic experimentation.

As a side detail, if we are able to navigate all entities in the hierarchy, we
are also able to search the hierarchy for specific components or matching
queries using `EntityQueryMask`.

### Querying for Hierarchy Status

We can design an `EntityQuery` that matches a specific hierarchy status. Below
is a table of what to query for each:

| Status                                | With                              | Without                          |
|---------------------------------------|-----------------------------------|----------------------------------|
| Has Parent                            | WorldTransform, RootReference     |                                  |
| Is Root (with or without descendants) | WorldTransform                    | RootReference                    |
| Is Root with Descendants\*            | WorldTransform, EntityInHierarchy |                                  |
| Is Standalone                         | WorldTransform                    | RootReference, EntityInHierarchy |
| No Transform                          |                                   | WorldTransform                   |

*\*Note: It is possible that all the descendants have been destroyed.*

### Writing Transforms Single-Threaded

Writing transforms single-threaded is not much different from reading the
`LocalTransform`, API-wise.

```csharp
public static void SetWorldTransform(Entity entity,
                                     TransformQvvs worldTransform,
                                     ref ComponentLookup<WorldTransform> worldTransformLookupRW,
                                     ref ComponentLookup<RootReference>  rootReferenceLookupRO,
                                     ref BufferLookup<EntityInHierarchy> entityInHierarchyLookupRO) => throw new System.NotImplementedException();
```

Of course, we can implement the full set of V1’s `TransformAspect` APIs using
this set of lookups. We can also provide both safe and unsafe shortcut variants
for things we’ve already looked up.

### Instantiating Entities with EntityManager

Usually when users want to instantiate an entity, they want to instantiate the
entire hierarchy. In this case, `EntityManager.Instantiate()` will do the right
thing out of the box. Because of how entity remapping works, the entire
hierarchy will be remapped correctly. All `WorldTransforms` are still up-to-date
as well. And there’s no external resources or cleanup components that need to be
interacted with. The entire hierarchy is ready to go!

The less-common use case of instantiating a child is far less simple. In this
case, the `RootReference` won’t remap. We will need a separate extension method
for this use case where after instantiating, it updates the parent hierarchy to
insert the new element. Let’s call this method `InstantiateChild()`.

### Destroying Entities

We’ve already established the rule for destroying non-root entities. Their
children (if any) get reparented to the next alive ancestor. So
`EntityManager.DestroyEntity()` just works.

For roots, things get a little tricky. In most cases, the descendants are
already in the `LinkedEntityGroup` and will be destroyed as well. However, if an
entity is parented at runtime or belongs to a subscene hierarchy without
children, then we might need a cleanup version. In this case, we have
`EntityInHierarchyCleanup`.

The question then is what do we do with it?

One answer is we could keep this component alive until all former descendants
are destroyed. This would require every `BufferLookup<EntityInHierarchy>` to be
complemented with `BufferLookup<EntityInHierarchyCleanup>` in all the previous
APIs. We’d also need a system to check each frame to test if the remaining
entities are still alive.

The second option is that we have a system update each frame to iterate
`EntityInHierarchyCleanup` on destroyed entities and correct all the remaining
orphans. This makes things tricky, because there may be a window between entity
destruction and this system which may still need to query the hierarchy. So
depending on how a project organizes the sync points,
`BufferLookup<EntityInHierarchyCleanup>` may or may not be needed.

The third option is to always require `LinkedEntityGroup` for roots, and have a
system that adds it for subscene pre-spawned entities. This is the cleanest
solution in isolation, but in practice could result in “fake parent constraints”
being a thing.

I’ve yet to decide which option is best.

### Changing Parents with EntityManager

With `EntityManager`, this one is fairly straightforward. We just need two
methods:

```csharp
public static void SetParent(Entity child, Entity newParent) => throw new System.NotImplementedException();
public static void ClearParent(Entity child) => throw new System.NotImplementedException();
```

Of course, we might want batching versions of the APIs for performance reasons,
in which we might have a `NativeArray` for each parameter.

## Performance Use Cases

We have sufficient APIs to express any kind of gameplay logic. However, not all
of these APIs are usable within high-performance contexts, such as parallel
jobs.

### Writing Transforms in Parallel

Writing transforms in parallel is tricky, because multiple entities in the same
hierarchy could be touched by different threads, which can cause race
conditions. There’s actually three scenarios to consider.

#### Root Exclusive Access and Only Iterating Roots

When we know that we have exclusive access to a root entity, and that exclusive
accesses are only ever promised for root entities, we can infer a lot of other
things.

The most common scenario for this is when iterating entities within an
`EntityQuery` that requires all entities within to be roots. In such a scenario,
we know that no other thread can ever be touching any descendant of a root we
are iterating. That means **all** the entities in the hierarchy are safe to
write.

Not just transforms, safe to write in general. We can write to any of their
components.

However, the only way V2 can guarantee these scenarios is via a custom job type
or Psyshock-style processor. Otherwise, this detail needs to be promised via an
unsafe API.

#### Root Exclusive Access but Not All Entities Are Roots

We may know we have exclusive access to a root entity, but another thread may
have access to other entities within the hierarchy. In this case, we at least
know that the other thread can’t be modifying the other entity’s transforms,
because it doesn’t have global write access to `WorldTransform` for all
entities. And `WorldTransform` has a rule that you never write to it directly.
But what about other components that don’t have this rule? Can we coordinate
which exclusive access is allowed to write to each component?

Here's a quick test, will this code throw?

```csharp
struct LookupA
{
    [NativeDisableParallelForRestriction] public ComponentLookup<WorldTransform> l;
}
struct LookupB
{
    [NativeDisableParallelForRestriction] public ComponentLookup<WorldTransform> l;
}

struct TestJob : IJob
{
    public LookupA lookupA;
    public LookupB lookupB;
    public Entity  entity;

    public void Execute()
    {
        if (lookupA.l.EntityExists(entity))
            lookupA.l[entity] = default;
        if (lookupB.l.EntityExists(entity))
            lookupB.l[entity] = default;
    }
}
```

With safety checks enabled, this code will indeed throw due to aliasing. That’s
actually really good news. Because it means that we can wrap component lookups
based on who we want to grant exclusive access. We might have a
`PhysicsComponentLookup`, and a `HierarchyComponentLookup`, and if they both
specify the same type, Unity throws. So really, this particular case isn’t all
that different from the one above in how we represent it.

#### No Exclusive Accessed

In this case, we simply can’t write immediately safely. The best we can do is
defer writes to a separate job. We’ll define a custom container for this:
`TransformDeferredWriter`. And users will have to schedule a job built into this
container after scheduling their jobs that write to it.

One benefit of deferring writes is that it no longer really matters much which
thread writes to which entity. Writing transforms to entities other than the
ones you are iterating is allowed, with a catch. We still need some scheme to
deal with conflicts. There are two solutions to this. One is that we check for
aliasing of commands, similar to how the Smoothie add-on does it. The other is
we use `sortKeys` to specify priority of commands. Perhaps both options should
be offered.

In case you are wondering about the performance of this, playback may likely be
composed of two jobs, one single-threaded job that organizes commands by the
chunks belonging to each root, and the other being a parallel job that processes
root chunks. For performance, I might use a target chunk per-thread linked list
for commands embedded in a stream, so that the single-threaded job is only
aggregating unique chunks identified by each thread. Don’t worry if that doesn’t
make any sense. All you need to know is that this case will be a little bit
slower than the previous two, and the results won’t be as immediately reflected.
But it should hopefully not be painfully slow.

### Instantiating Entities via Command Buffers

This is probably the weakest area of V2’s design, because command buffers
directly conflict with one of V2’s guiding principles.

For a command buffer, anything can happen between recording and playback. And
the hope is that the commands are still meaningful at playback time. If not, bad
things happen, including crashes. New DOTS users who aren’t careful about this
often get bit really hard by bugs and end up compromising their sanity. You
might have been one of them at some point (or maybe you still are).

Instantiating an entity is pretty safe, but we have no way of knowing what will
happen to the reference hierarchy between recording and playback. And that makes
setting up a spawn position (which affects all entities in the hierarchy) at
record time **very difficult**.

There are two tools for this. The first is custom command buffers. And the
second is a `DynamicBuffer` on the `worldBlackboardEntity` that we append to
with deferred spawn positions. However, that’s just for the spawn point. If we
also need to parent entities, then things get even more complicated.

I’m going to need some time to figure out what is the best option. I wonder if
perhaps this could be solved by a generalized modular command buffer, assuming I
could make such a thing outperform `EntityCommandBuffer`?

If you have suggestions, feel free to leave them. Just be warned, I’m going to
challenge you to think deeply about edge cases.

## Special Use Cases

There are still a few special use cases and notes worth highlighting.

### Ticking

Ticking is a new feature planned for supporting fixed-rate simulation steps,
which is especially useful for networking. The plan for ticking in V2 is to have
separate Ticking versions of all components and APIs. I’m more comfortable with
this because the V2 archetypes are generally smaller than V1, and with
everything being up-to-date at all times, interpolation systems won’t have too
much to worry about.

### Sockets and Exposed Skeletons

Currently, Kinemation uses the explicit transform hierarchy syncs to inject
socket data from optimized skeletons into the transform hierarchy. With V2,
sockets will have to be explicitly dispatched.

For exposed skeletons, it should be possible to provide batch APIs for writing
multiple transforms in a hierarchy in an immediate context, so there shouldn’t
be any regressions there.

### Reparenting in Jobs

Not all reparenting operations require structural changes. I’m not sure how
common this will be, but in cases where it is known in advance that structural
changes aren’t required, it may be possible to perform reparenting in a job.
Moving an entity to a different part of the hierarchy is one use case where this
holds true.

### Expirables

The rules for expirables may have to change, to discourage roots from being
destroyed before their children. Expirables current mess with
`LinkedEntityGroup`, which V2 will rely upon to ensure entities don’t lose their
hierarchies.

### Unika

Unika will likely want a dedicated script resolver that has hierarchical scope
that can run in parallel. This may introduce a new dependency.

### Abstract Transforms

Abstract Transforms will lose `IAspect` support, and will need to be replaced
with custom types. This means compatibility with `IJobEntity` and idiomatic
foreach won’t be very good, if possible at all.

Aside from that, V2 transforms have much stricter rules than Unity Transforms
for writing. So abstract writers will generally have API patterns that conform
towards QVVS V2 rules. This means there may be some performance regressions for
Unity Transforms compatibility. I hope these won’t be that significant.

## Open To Feedback

Are there use cases that were missed? Are there areas of V2’s design you would
like to be further discussed? Do you think there are some other aspects that
should be considered? Please reach out on the Latios Framework discord server!

This document may be updated with further design changes and improvements.
Nothing here is final.
