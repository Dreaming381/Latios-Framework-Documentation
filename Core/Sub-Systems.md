# Sub-Systems

A `SubSystem` is a subclass of `SystemBase` which exposes additional API and
boilerplate reduction mechanisms within the framework. You can use them exactly
as you would a `SystemBase`, and frequently you will want to do just that.
However, there are some additional features to take advantage of.

Nearly all `SubSystem` features are [extended](ISystem%20Support.md) to
`ISystem`. `ISystem` is the preferred system type for performance. `SubSystems`
are only allowed in a `LatiosWorld`, which currently does not support baking and
streaming worlds.

## Fluent Queries

[Fluent query](Fluent%20Queries.md) expressions can be initiated using the
member property `Fluent`. You will likely want to use this inside of
`OnCreate().`

## Blackboard Entities

The two [blackboard entities](Blackboard%20Entities.md) can be accessed through
the member properties `sceneBlackboardEntity` and `worldBlackboardEntity`.
However, you must assign them to local `Entity` variables before using them in a
Bursted lambda job.

## Collection Component Dependency Management

When fetching a [collection
component](Collection%20and%20Managed%20Struct%20Components.md), rather than
complete the `JobHandle`s associated with that component, the `JobHandle`s will
be combined with `Dependency`. Additionally, by default `Dependency` will
automatically be used to update the `JobHandle`s of the collection component
after the `SubSystem` finishes `OnUpdate()`. When combined with lambda jobs, you
may never have to explicitly touch `Dependency` nor any other `JobHandle`.

## LatiosWorld Sync Point Dependency Management

The [LatiosWorld](LatiosWorld%20in%20Detail.md) instance can be accessed through
the `latiosWorld` member property without casting. From there, the `latiosWorld`
provides access to the
[SyncPointPlaybackSystem](Custom%20Command%20Buffers%20and%20SyncPointPlaybackSystem.md)
via the `syncPoint` property. Once you do this, `Dependency` will automatically
be sent to `SyncPointPlaybackSystem` after the `SubSystem` finishes
`OnUpdate()`. When combined with lambda jobs, you may never have to explicitly
touch `Dependency` nor any other `JobHandle`.

## New Scene Callback

When using the scene management system, a `SubSystem` can receive a callback
when a new scene is created by overriding `OnNewScene()`. The order these
functions are called do not reflect the system order. This callback happens
after the subscenes have been loaded and the blackboard entities have been
merged.

## Custom Update Criteria

`ShouldUpdateSystem()` can be overridden if a [custom
criteria](Super%20Systems.md) should dictate whether the `SubSystem` should
execute `OnUpdate()`. This does not disable Unity’s `EntityQuery` checks. To
disable those, add `[AlwaysUpdateSystem]` to the `SubSystem`.

*Caution: Dependency and version numbers have not been updated yet when this
method is invoked. Only EntityManager, EntityQuery, and BlackboardEntity
operations are recommended.*

## Disabling Multi-Update Sync Points

Unity ECS systems are designed to always sync the jobs they scheduled the
previous update. This prevents multiple frames of jobs being queued up and never
completing. However, for systems that may update multiple times per frame, such
as those which update in `FixedRateSimulationSystemGroup`, these sync points can
hurt performance.

In a `LatiosWorld`, you can add the `[DontSyncPreviousUpdatesThisFrame(int
maxUpdatesWithoutSync)]` attribute to any system. This will cause the system to
skip syncing after the first time it updates in a frame, up to the maximum
number of updates.

**Warning:** *The Dependency property is not guaranteed to account for all jobs
scheduled in the previous update. It only examines component read-write
accesses, and treats the previous update’s Dependency value as coming from a
different system. This can cause safety errors if your system only reads
components, but writes to a persistent collection stored on the system via a
job.*
