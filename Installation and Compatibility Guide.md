# Latios Framework Installation and Compatibility Guide

This guide will walk you through installing Latios Framework into your DOTS
project. It will also show you how to enable or disable features of the
framework. This guide is specifically designed to address custom project needs.
For the basics of Latios Framework setup, it is recommended you review [Core:
Getting Started](Core/Getting%20Started.md) first.

## Installation

When you first install the package, you may experience compiler errors due to
lack of special scripting defines. Add the requested scripting defines to
continue.

Latios Framework 0.11 uses a custom transform system rather than Unity’s
Transforms by default. This system will bake `GameObject` `Transform`s fine, but
may pose compatibility issues with other ECS packages. If compatibility is a
larger concern to you than the performance and feature advantages of this custom
transform system, you can enable a compatibility mode for Unity Transforms via
the LATIOS_TRANSFORMS_UNITY scripting define. Note that Unity Transforms is
required for full NetCode support as QVVS support is experimental. Additionally,
Unity Transforms are required for Unity Physics unless you modify the Unity
Physics package.

Nearly all Latios Framework functionality requires that the World instance be a
subclass instance called `LatiosWorld`. If your project currently uses default
world initialization, you simply have to go to your project window, *right
click-\>Create-\>Latios-\>Standard Bootstrap – Injection Workflow*. This will
create an instance of an `ICustomBootstrap` which sets up a variation of the
default world, but with Latios Framework components installed.

### Installing with Existing ICustomBootstrap

If you are already using an `ICustomBootstrap`, you may still be able to install
Latios Framework. The following lines in `Initialize()` are the minimum
requirements to create a working `LatiosWorld`. You can either assign this world
to World.`DefaultGameObjectInjectionWorld` or create multiple `LatiosWorld`
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

Currently, out-of-the-box is supported for the following:

-   Windows
-   Linux Desktop
-   Mac OS
-   Android (including Meta Quest headsets)

In addition, if building for IL2CPP, you may need to set the code generation
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
