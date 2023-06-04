# Automatic Dependency Management Errors

Some features of the Latios Framework such as collection components and the
smart sync point provide automatic dependency management of containers for jobs.
However, this process is not foolproof, and errors can result when not used
correctly. This document describes the processes, errors, and resolutions.

## How Automatic Dependency Management Works

The basic idea behind automatic dependency management is that the
`LatiosWorldUnmanaged` records accesses to collection components by a system
during the system’s update. Then when the system finishes, the
`LatiosWorldUnmanaged` captures its `SystemState.Dependency` and applies it to
its internal dependency storage.

To do this, special internal `LatiosWorldUnmanaged` methods must be called
before and after every system update. The method before the update checks for
errors and then adds the system to the stack of currently executing systems. The
method after the update then records the `SystemState.Dependency` and processes
all the dependencies. Then it pops the executing system off the stack.

These special systems are called automatically by the following:

-   Directly in the `ComponentSystemGroup` `OnUpdate()`
    -   SuperSystem
    -   RootSuperSystem
    -   FixedSimulationSystemGroup
    -   LatiosWorldSyncGroup
    -   PreSyncPointGroup
-   Via Custom IRateManagers
    -   InitializationSystemGroup
    -   SimulationSystemGroup
    -   PresentationSystemGroup
-   Before and after `OnNewScene()` callbacks
-   Manually via user via `SuperSystem.UpdateSystem()`

Prior to the first of these methods being called, all accesses to collection
components and the sync point are untracked. This allows setup in `OnCreate()`
on the main thread to work without any extra overhead.

Accesses in all other contexts are tracked via `LatiosWorldUnmanaged`, but a
`JobHandle` will not be automatically fetched for these dependencies. It is up
to the user to provide `JobHandles` in these contexts, or else errors will
occur.

## Methods to Update Accesses Manually

If you ever need to manually specify a `JobHandle` or declare a main thread
access, use these methods:

-   LatiosWorldUnmanaged.syncPoint.AddJobHandleForProducer()
-   LatiosWorldUnmanaged.syncPoint.AddMainThreadCompletionForProducer()
-   LatiosWorldUnmanaged.UpdateCollectionComponentDependency\<T\>()
-   LatiosWorldUnmanaged.UpdateCollectionComponentMainThreadAccess\<T\>()
-   BlackboardEntity.UpdateJobDependency\<T\>()
-   BlackboardEntity.UpdateMainThreadAccess\<T\>()

## Common Access Context Errors and How To Fix Them

There are several different contexts where the `LatiosWorldUnmanaged` might be
accessed that could cause problems. Access in these locations aren’t necessarily
incorrect, but they may require extra steps to prevent errors.

### Accessing in a MonoBehaviour or Other Non-ECS Location

Naturally, the Latios Framework has no way of knowing what jobs if any are
scheduled in these contexts. Therefore, the solution is to always provide a
`JobHandle` or declare that the access was exclusively performed on the main
thread.

### Accessing in an Untracked System

If you get an error inside the update of a system, it may be that the
`LatiosWorldUnmanaged` doesn’t know that the system is executing in its stack.
This can easily happen if you update your system in a custom
`ComponentSystemGroup`. In such situations, prefer to change that
`ComponentSystemGroup` into a `SuperSystem`. If the system is a Unity or
third-party system, put your custom system inside a `RootSuperSystem` inside the
`ComponentSystemGroup`.

### Accessing Inside of a System that Updates Other Systems

A system that accesses collection components or the sync point must finish
executing before the next system starts running, or else the cleanup process
won’t have a chance to run. In these situations, you will have to provide the
`JobHandles` or declare access was exclusive to the main thread manually.
