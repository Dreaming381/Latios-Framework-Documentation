# Latios Framework Installation and Compatibility Guide

This guide will walk you through installing Latios Framework into your DOTS
project. It will also show you how to enable or disable features of the
framework.

This guide is specifically designed to address custom project needs. For the
basics of Latios Framework setup, it is recommended you review [Core: Getting
Started](Core/Getting%20Started.md) first.

## Installation

When you first install the package, you may experience compiler errors due to
lack of special scripting defines. Add the requested scripting defines to
continue.

Some features of the framework are no longer compatible with Unity 6.3 and
newer. Add the scripting define LATIOS_DISABLE_CALLIGRAPHICS for these editor
versions. This issue will be resolved in a future framework release.

Latios Framework 0.14 uses a custom transform system rather than Unity’s
Transforms by default. This system will bake `GameObject` `Transform`s fine, but
may pose compatibility issues with other ECS packages. If compatibility is a
larger concern to you than the performance and feature advantages of this custom
transform system, you can enable a compatibility mode for Unity Transforms via
the LATIOS_TRANSFORMS_UNITY scripting define. Note that Unity Transforms is
required for full NetCode support as QVVS support is experimental. Additionally,
Unity Transforms are required for Unity Physics unless you modify the Unity
Physics package.

Nearly all Latios Framework functionality requires that the `World` instance be
a subclass instance called `LatiosWorld`. If your project currently uses default
world initialization, you simply have to go to your project window, *right
click-\>Create-\>Latios-\>Standard Bootstrap – Injection Workflow*. This will
create an instance of an `ICustomBootstrap` which sets up a variation of the
default world, but with Latios Framework modules installed.

### Installing with Existing ICustomBootstrap

If you are already using an `ICustomBootstrap`, you may still be able to install
Latios Framework. The following lines in `Initialize()` are the minimum
requirements to create a working `LatiosWorld`. You can either assign this world
to `World.DefaultGameObjectInjectionWorld` or create multiple `LatiosWorld`
instances in a multi-world setup.

```charp
var world = new LatiosWorld(defaultWorldName);
world.initializationSystemGroup.SortSystems();
return true;
```

`LatiosWorld` creates several systems in its constructor. This can throw off
`DefaultWorldInitialization.AddSystemsToRootLevelSystemGroups()`. You may need
to remove some system types from the list. `BootstrapTools.InjectSystems()`
often avoids this problem but otherwise produces the same results.

If you do use your own bootstrap, you should still create a bootstrap template
and delete the class named `LatiosBootstrap` which implements
`ICustomBootstrap`. The Latios Framework defines and uses custom bootstraps for
editor and baking contexts, so performing this action will ensure you have
those.

## Managing Features

Features are controlled through the use of *installers*. You can see these
installers in action by looking through the bootstrap templates. Every bootstrap
is found in a static class named `<moduleName>Bootstrap` or
`<moduleName>BakingBootstrap`. Check the documentation of each method to learn
which bootstraps it needs to be called within.

## Platform Support

Latios Framework does not support all platforms out-of-the-box. This is because
the Kinemation module ships with a native plugin which is currently only built
for those platforms. You can learn more about the plugin and how you can help
extend it to work on more platforms
[here](https://github.com/Dreaming381/AclUnity).

Currently, the Latios Framework has full out-of-the-box support for the
following platforms:

-   Windows
-   Linux Desktop
-   Mac OS
-   Android (including Meta Quest headsets)

Other platforms are possible, but will require compiling a required [native
plugin](https://github.com/Dreaming381/AclUnity) for the platform. You can also
add the scripting define `LATIOS_DISABLE_ACL` to disable all usage of the native
plugin if your project doesn’t need it and you’d like to target other platforms.

In addition, if building for IL2CPP, you might need to set the code generation
mode to *Faster (Smaller) Builds*.

If there is some other unexpected behavior, that is likely a bug. Please report
the issue!

### IL2CPP Specifics

Latios Framework uses reflection at runtime for a small number of features. The
following lists the areas where reflection is used on otherwise inaccessible
members. They should be correctly preserved, but if not, please report them.

-   Core
    -   Any `ICollectionComponent` or `IManagedStructComponent` implementing
        type will have generated code which will need to be preserved.
-   Psyshock
    -   Psyshock calls `EarlyJobInit()` on generic jobs for
        `IFindPairsProcessor` and `IFindObjectsProcessor`. The `EarlyJobInit()`
        method may accidentally be stripped for these jobs.
-   Unika
    -   All Unika types and interfaces that are handled by codegen need to be
        preserved.

## Cross-Platform Determinism

The Latios Framework experimentally supports Burst’s deterministic floating
point mode for the following modules:

-   Core
-   QVVS Transforms
-   Calci
-   Psyshock
-   Unika
-   Kinemation (partial)

You can enable this mode using the scripting define LATIOS_BURST_DETERMINISM.

**This does not mean using the Latios Framework promises cross-platform
determinism!** That isn’t even promised by Unity’s ECS packages. You will need
to perform your own analysis.

For Kinemation specifically, optimized skeleton maintenance systems are
deterministic. That is, any built-in systems which affect the transforms of
bones, sockets, or transforms are deterministic. Systems which calculate
`WorldRenderBounds` and `RenderVisibilityFeedbackFlag` are NOT deterministic.
