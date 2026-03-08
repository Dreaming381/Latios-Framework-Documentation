# Long-Term Risks of the Latios Framework

The Latios Framework project began in 2018. If you compare that to today, you
can see that this project has been active for quite a while. As long as Unity
remains a viable platform to achieve the goals of this framework, then
development will continue.

With that said, Unity changes over time. Sometimes those changes are good.
Sometimes they are bad. And sometimes they are good for non-framework users, but
have really bad consequences for this framework and other 3rd-party extensions.

This document outlines current and future risk factors that originate from
Unity’s changes. These are not to discourage you from adopting the framework for
your project, but rather things you should advocate for publicly and privately
to Unity, as otherwise no one can have the nice things the framework offers.

## Entities Graphics – GRD Consolidation

**Risk:** High

**Consequences:** Fork of packages required

Entities Graphics communicates to the Unity Engine through a special API surface
known as the `BatchRendererGroup`. This API is what allows entities to be
efficiently translated into instanced draw calls sent straight to the GPU. This
API was so effective, that Unity decided to build a similar solution using this
API, but for `GameObjects`. That solution became known as the GPU-Resident
Drawer, or GRD.

Then Unity migrated or laid off some of the Entities Graphics team. And then the
others left. And now none of the original authors remain. Instead, one
individual responsible for GRD was left to build a new team to take over both
solutions.

While initially GRD was built exclusively on the `BatchRendererGroup` API, this
API wasn’t sufficient for the features Unity wanted to build. Rather than extend
the API, Unity started directly calling into GRD methods from the Scriptable
Render Pipeline. This created feature fragmentation.

Kinemation uses the `BatchRendererGroup` API. Thus, it does not have access to
these special GRD features like GPU occlusion culling. However, GRD is
architected in a way that fundamentally prevents it from implementing key
Kinemation features and optimizations. If GRD were to support all of
Kinemation’s rendering features will similar performance, then this would be a
non-issue. But that is very unlikely to manifest.

With the consolidation of Entities Graphics and GRD, the GRD codebase is going
to be expanded to support rendering entities. The details of how it will do this
is not public at the time of writing. But this decision further incentivizes the
trend towards GRD being the only way to draw things with DOTS instancing
shaders, and that’s a major risk to Kinemation. If Unity continues in this
direction, the framework would need to fork the core Scriptable Render Pipeline
packages to customize things to meet Kinemation’s needs.

## Unified Transforms

**Risk:** Medium

**Consequences:** Fork of packages required

Similar to the above, Unity is also trying to consolidate the `GameObject` and
ECS transform systems. This is handled by the engine providing internal access
to the Entities package to access its internal transform system. Consequently,
this means that the framework cannot access this system without a fork of the
package.

Whether access to the system is required is another matter. At runtime, projects
fully committed to QVVS transforms require very few `GameObjects`. And often, it
is beneficial to explicitly sync `GameObject` and entity transforms at specific
points in the frame to avoid the engine forcing ECS job chains to complete (the
engine likes to complete transform-touching jobs in quite a few places in the
frame, especially for the camera).

However, if baking and lifecycle management become too closely tied to Unity’s
unified transforms implementation, then things become problematic. But
presently, the only hint that this is even a possibility is the concept of
unification, in which entities would behave more like `GameObjects`. And
currently, baking and entity lifecycle management is very different from
`GameObjects`. But this is very speculative, hence the risk is lower.

## MSBuild

**Risk:** Medium

**Consequences:** Fork of packages required

Unity’s push to CoreCLR includes replacing the build system. The only problem?
Currently, the framework relies heavily on asmref files to make package
internals and the engine hooks accessible to custom code, because Unity has
failed to provide sufficient APIs for more complex needs. MSBuild could mean
that asmref goes away, likely without a replacement. This would immediately
necessitate forks of all packages.

A number of DOTS users even outside the framework community heavily rely upon
asmref. So Unity is going to make a lot of people upset if they don’t provide an
alternative. Hence the risk is lower.

## NetCode Compatibility

**Risk:** High

**Consequences:** Lack of future NetCode support, even in Unity Transforms mode

Unity staff have expressed the intent to subclass the World type for NetCode.
Currently, the Latios Framework already subclasses the World type via
`LatiosWorld`. But this type is also needed for editor worlds and local
simulation worlds, so it may not be trivial to make `LatiosWorld` subclass the
NetCode subclass instead under conditional compilation.

If this were all to play out this way, then there would be a significant amount
of work required for the framework to eliminate `LatiosWorld`. And that work
simply cannot be justified for solo development. Without major community
contributions to help preserve compatibility, the framework is likely to drop
NetCode support.

There are plans for the framework to provide its own networking solution in the
future.

## Source Generators and IAspect Removal

**Risk:** It is happening right now

**Consequences:** The Latios Framework is no-longer beginner-friendly (beginning
0.15)

Sometimes, the framework needs to internally store data in complex ways. This is
complexity users shouldn’t need to bother with. Historically, the framework has
provided user-friendly abstractions. However, `IJobEntity` and `SystemAPI` don’t
support abstractions. They used to support one specific form of abstraction,
which was `IAspect`, but Unity decided to kill that. That means there’s no way
to make easy-to-use framework APIs compatible with the easy-to-use ECS APIs.

A potential solution would be for the framework to provide its own source
generator alternatives for such things. But that’s a lot of work that doesn’t
help the framework reach its desired feature set any faster. So it may be quite
a while before something like that happens, unless the community pitched in to
make it happen (unlikely).

## API Evolution Sluggishness

**Risk:** It is happening right now

**Consequence:** The Latios Framework is no-longer inert by default (beginning
0.15)

The Latios Framework has always had the policy that if you were to install it in
an existing project and gotten it to compile, you would experience no change in
project behavior until you added a bootstrap. However, Unity has been
ridiculously slow at implementing APIs and workflows that allow for
extensibility. And as such, the framework has had to hook into unconventional
places to enable fundamental features. As a result, it has become unsustainable
for the framework to follow this policy. If you install the framework, some
behavior might change due to some features always being enabled.
