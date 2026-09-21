# Getting Started with Latios.Core Framework

## Runtime Settings

In *Project Settings*, under the *Player* tab, in *Other Settings* and under the
*Configuration* header, set the *API Compatibility Level* to *.NET Standard*.

Also, add the Scripting Define Symbols:

-   UNITY_BURST_EXPERIMENTAL_ATOMIC_INTRINSICS

### Mono

No additional settings required.

### Il2Cpp

In the *Build Settings*, you may need to set the *IL2CPP Code Generation* to
*Faster (smaller) builds*

## Bootstrap

Latios Framework requires a custom bootstrap to inject its custom features into
the ECS runtime.

You can create one from one of the templates using the Project folder’s create
menu (or the top menu’s “Assets” tab) and selecting *Latios-\>Bootstrap*.

If you want the broadest compatibility with Unity’s ecosystem, then pick the
Unity Transforms – Injection Workflow. If you want to go all-in on the Latios
Framework’s features and workflows, then pick *Standard - Injection Workflow* or
*Standard – Explicit Workflow*.

Injection workflow is Unity’s ECS workflow, where systems are automatically
created and ordered with attributes. Explicit workflow will instead inject Unity
systems, but then allow you to inject only top-level systems which set up
children systems in a top-down manner. This is often the preferred workflow for
experienced users. See [Super Systems](Super%20Systems.md) for more info on this
workflow.

If you use a Unity Transforms bootstrap, you will need to add
LATIOS_TRANSFORMS_UNITY scripting define to your project.

**Warning:** If you have the Unity Physics, Character Controller, or Vehicles
package installed, those packages are only compatible with Unity Transforms mode
of the framework, and you must use a Unity Transforms bootstrap. QVVS
alternatives for some of these packages may be found in the Latios Framework
Add-Ons package.

For NetCode Projects (Note: NetCode support is deprecated and will no longer
work in a Unity version later than 6.3), the Latios Framework will detect the
NetCode package and enable additional C\# files for compilation. One of these
provides a dedicated NetCode bootstrap type called
`LatiosClientServerBootstrap`, which offers Latios Framework equivalents of
`ClientServerBootstrap`. You can further customize each type of world with the
additional bootstrap interfaces provided, which you can automatically generate
from the *NetCode Standard – Injection Workflow* bootstrap. The *NetCode QVVS –
Explicit Workflow* bootstrap is experimental.

After the bootstrap is created, it can be
[customized](Customizing%20the%20Bootstraps.md).

## Project Structure Tips

### Only one scene active at a time (When Scene Management is installed)

I know, I know. You’ve always had multiple scenes loaded at runtime because of
conflicts and editor performance and all that. But with subscenes, there’s no
need for that anymore. You can have as many subscenes as you like, but you
should keep scenes separate so that when you swap scenes, the slate can be wiped
clean.

### Keep Systems in Separate Assemblies from Components

This isn’t a hard-and-fast rule, but some of the Latios Framework custom
component types rely on source generators, and this can confuse `SystemAPI` if
they are in the same assembly as the systems that use them.
