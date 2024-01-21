# Upgrade Guide [0.8.6] [0.9.0]

Please back up your project in a separate directory before upgrading! You will
likely need to reference the pre-upgraded project during the upgrade process.

This document only highlights the common things to be aware of when upgrading.
See the changelogs of each module for a complete list of changes.

## Core

### New Unity Packages

All Unity ECS Packages have been updated to 1.1.0-pre.3 versions. You will need
to upgrade these packages, which may introduce some behavioral differences.

### New Bootstrap and Installers

Bootstraps have changed. You will likely want to recreate your bootstrap from
the one of the templates.

### Fluent Queries

Fluent Queries have been completely redesigned to properly account for
`IEnableableComponent`. In addition, the new design always applies “weak” rules
from prior releases. Most of the time, you will want to replace `WithAll<>()
`with `With<>()`. But keep in mind that `With<>()` does not consider enabled
states.

## Kinemation

### Mecanim Moved Out

All Mecanim functionality has been moved out and into the Mimic module as an
Addon. You will need to adjust namespaces, asmdefs, and bootstrap initialization
to compensate.

### Multiple Materials Per Entity

By default, entities are now baked with combined materials onto one or two
entities (two only if the object has both opaque and transparent materials). You
will likely need to alter your logic for handling material property overrides
for meshes with multiple materials. Additionally, if you used the override
renderer baking features, you will need to rewrite these to work with a new
unified API.

## New Things to Try!

### NetCode Support

NetCode support is back, but only for Unity Transforms for now. Only the Core
module is remotely aware of the changes. But you can still use the other modules
for presentation or other purposes.

### Core

Core got a lot of improvements. `IAutoDestroyExpirable` presents a new generic
paradigm for negotiating the destruction time of entities to prevent premature
destruction. `ICollectionAspect` allows exposing parts of collection components
in unique ways. Blackboard entities will now merge enabled states. And all
runtime `ComponentSystemGroups` support updating framework-aware systems.

### Psyshock Physics

Two new namespaces `UnitySim` and `XpbdSim` bring some new experimental
features. The former allows generating contact points between two colliders,
while the latter allows evaluating distance and tetrahedral volume constraints.

### Kinemation

Kinemation received a ton of new features such as the `GraphicsBufferBroker` and
`SkinningAlgorithms` which open up the rendering pipeline for user
modifications. Kinemation used to supply a backend surface for Calligraphics.
But now Calligraphics uses these public APIs instead to perform its own custom
rendering.

### Mimic

New module!

Actually, this module is the new home for the Mecanim runtime. Along with lots
of fixes, this runtime adds experimental support for blend shape animation with
the LATIOS_MECANIM_EXPERIMENTAL_BLENDSHAPES scripting define.
