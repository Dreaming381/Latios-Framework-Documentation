# Level Up Your ECS - Baking (System) Recipes

Hi everyone!

More and more, I have been told by people that they still feel too new to
understand some of the technologies I build or aren’t aware of some the ECS
tools I use. Even those with experience and competence have asked me about
advanced topics and tools that are available but no one really dives into. It is
time I share some of this information with the community, in the hopes that
there will be more people doing more advanced things and sharing them online.
While I will reference the Latios Framework several times as examples, none of
what I describe in these series is exclusive to that. Rather, this info applies
to anyone working with Unity's ECS. These series are meant for intermediate and
advanced users, and assume some minimal level of understanding of Unity's ECS.

This series is about baking systems. As I have stated several times across
various channels, baking systems are really hard to get right. Unfortunately,
messing up is hard to detect because Unity provides no correctness checks over
this mechanism. Instead, you'll just be stuck with these mysterious bugs that
seem to correct themselves by closing and reopening the subscene. So for this
first post, I am going to break down what is happening, what causes these
mysterious bugs, and what to watch out for. By the end, you'll realize that
baking systems are hard, and even Unity themselves frequently write them
incorrectly.

## When does baking happen?

Disclaimer, I am not Unity, and Unity may choose to change this behavior in the
future.

### Opening a subscene

When you first open or create a subscene, Unity builds up an internal hierarchy
data structure of all `GameObjects` and `Components` in the subscene. It also
constructs a dedicated ECS `World` for all the entities associated with that
subscene.

Next, it runs a Pre-Baking System Group. Unless you are modifying the Entities
package directly, this group is likely not helpful, and I won't mention it
anymore.

Then, for every component on every `GameObject`, it runs all compatible bakers.
Bakers record their changes via an `EntityCommandBuffer`. Once all the bakers
run, the ECB is played back.

After that, there's the `TransformBakingSystemGroup`, followed by the normal
`BakingSystemGroup`.

Then a single system updates `LinkedEntityGroup`, followed by Companion
`GameObject` baking.

Then `PostBakingSystemGroup` runs.

And finally, a `BakingStripSystem` runs to clean up any `[TemporaryBakingType]`
components.

From now on, I will refer to all the steps so far after running the bakers as
"the baking system groups".

The last thing that happens is that the subscene entities get copied into the
Editor World using a diff copy mechanism (which I will refer to as “diff-merge”
from now on). I won't explain the details here, but changes caused by the main
Editor World are purely for preview and those changes do not propagate backwards
into the baked subscene.

### Incremental Updates

The subscene is now open, and baking is now operating in "incremental mode".
This mode is both what allows you to see the effects of your changes
immediately, and is also the source of all the frustrating restrictions that
come with bakers, and all the bugs that people write in baking systems.

For every authoring component type, Unity keeps a collection of bakers that run
on it. These bakers are static and run for all subscenes. However, there is a
unique baker state per authoring component instance per baker. This state keeps
track of all the other authoring components and assets the baker declared as
dependencies the last time it ran on the authoring component instance.

Whenever you make a change to any authoring component, Unity reruns all bakers
associated with that component as well as any bakers on any components that
declared a dependency on the changed component. "Rerunning" actually means first
playing back an `EntityCommandBuffer` that "undoes" whatever the baker did for
that authoring component the last time. Then, it does normal baker execution.
The number of `Baker.Bake()` invocations that run in incremental baking with
every edit should be significantly smaller than the full rebake when the
subscene was opened.

After the bakers run, the baking system groups run in their entirety over all
the entities in the subscene's baking world. And then this world is diff-merged
into the Editor World.

This process repeats every time the user makes an edit to the authoring
representation of a subscene.

### Closing the subscene

Closing the subscene works a bit surprisingly, in that Unity discards the
incremental subscene world state completely. Instead, just like opening, it does
a full rebake from scratch with all the bakers, but does a few extra steps as
well. Also, in Prerelease 15, this baking happens in a completely separate
background Unity process without Burst.

So after all the bakers and baking system groups run, there's another set of
systems that run in the category of `EntitySceneOptimizations`. These systems
all get dumped into a `ComponentSystemGroup` called `OptimizationGroup`. The
group runs **twice**. The idea being that the first time it makes structural
changes and the second time it updates chunk component values. I'm not sure I
agree with that philosophy versus having explicit groups with the latter issuing
a warning if there were structural changes detected, but that's how Unity does
it today.

And then finally, the subscene gets serialized, remapping blob assets in the
process.

## Baking System Basics

I said that Bakers make all their changes via ECBs. There is one exception.
Bakers create entities directly, and they always create the entities with one of
several initial archetypes. If the `GameObject` is a prefab, the entity receives
the `Prefab` tag. If the `GameObject` is disabled, the entity receives the
`Disabled` tag. These are inherited by any additional entities created via
`CreateAdditionalEntity` inside a baker.

Baking systems are like any runtime systems, in that they make entity queries,
and process entities. You can make them Burst-compiled `ISystem` types for a
more responsive editor experience. But consequently, queries will likely not run
on the entities you care about because queries ignore `Disabled` and `Prefab`
entities by default. Therefore, your baking systems should always query
`Disabled` and `Prefab` entities via `EntityQueryOptions`. This is a really
common mistake to make, so always check that first when a baking system doesn't
work right.

**Do not subclass BakingSystem.** This is not a base class system, but rather a
system for managing the state of bakers. Fetch a reference to it to get the
BlobAssetStore. And add
`[WorldSystemFilter(WorldSystemFilterFlags.BakingSystem)]` to an otherwise
normal system to create a baking system.

Also, because baking systems run on the full set of entities in the subscene, be
cognizant of the work you are doing. Take advantage of change filters to skip
chunks. And be wary of structural changes. While it isn't as bad as runtime, you
can still make a horrible editor experience with a poorly-written baking system.

There are two special attributes that you can decorate on components,
`[BakingType]` and `[TemporaryBakingType]`. The former makes it so that the
component type is removed from all entities during diff-merge or subscene
serialization. The latter gets stripped at the end of the baking system groups
every time they run. You can store unstable data in `[TemporaryBakingType]` such
as `UnityObjectRef` (a handle to a `UnityEngine.Object` that can be used in
unmanaged code such as a key to a `NativeHashMap`) or `InstanceID`s. If there
are important pieces of information a baking system needs about the authoring
world, there are ways to transfer it using `TemporaryBakingType`.

## Incremental Baking System Problems

Bakers record every change they make so that prior to being rerun, they can undo
those changes.

Unfortunately, baking systems do not have this luxury. This means it is totally
up to **you** to undo any changes when an entity is rebaked.

Let's walk through an example. Let's suppose you have a `TeamColorAuthoring`,
which gives you the option of setting a color. The baker sets a `TeamColor`
`[BakingType]` to the entity. Because a baker isn't allowed to add components to
the children, a baking system is written for that purpose. The baking system
iterates through all children of an entity with the `TeamColor` and adds a
material property color component to those children. Seems simple, right?

Now a user adds the `TeamColorAuthoring` to a root `GameObject`, and in the game
tab, all the children change to that color. The user tweaks the
`TeamColorAuthoring`'s color value and the children update in realtime. So far
so good.

Then the user removes the `TeamColorAuthoring`. The colors on the children stay,
because the material property components were added by a baking system and there
was no baking system logic to ever remove them. Okay, that's only a little
confusing. The component is destructive in nature. People write `MonoBehaviours`
to work like that too sometimes.

But then, when the subscene is closed, the team colors disappear. And now the
designer is really confused. Cue bug reports, frustrated programmers, and
shouting matches between two big egos belonging to two very different
disciplines. It ain't pretty. And it is all too easy to do in the current
design.

These leftover "phantom components" aren't the only kinds of residual artifacts
that can occur with incremental baking. But they are **by far** the most common
and also quite tricky to catch and fix in a performant manner. Always test for
this by adding and removing an authoring component and make sure the subscene
entities don't have any leftover components.

In future posts, I will go over some recipes I have developed to do some common
operations that require baking systems, while simultaneously avoiding baking
system artifacts. It is my belief that if you follow these recipes faithfully,
you will be able to avoid many conflicts with your designers.

In the meantime, I would like to open this thread up for discussion and
questions.
