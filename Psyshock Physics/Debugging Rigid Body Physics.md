# Debugging Rigid Body Physics

Over the past few months, many people have been experimenting with the
`UnitySim` API directly, or indirectly through the Anna physics add-on. And many
bugs have been reported. Most of the bugs have to do with rigid body collisions
not behaving in intuitive ways. Unfortunately, the underlying issue is
deep-rooted, and difficult and time-consuming to debug and fix.

This document will first cover how to effectively identify and report such bugs.
Then it will provide some background as to why these issues are happening. And
finally, it will provide some starting points into how you can help combat these
issues if you believe in Psyshock’s mission.

## What to Look For

The issue usually manifests itself as “surprise energy”. An example might be a
dropped cube that was tumbling about on the ground and nearly coming to a rest,
when all of a sudden, it “pops up” again. Maybe if flew off into the atmosphere,
or maybe it was just a little nudge. But either way, it doesn’t make any sense.

While it might seem like this is an issue with contact resolution, this is
almost always an issue with `Physics.DistanceBetweenAll()` and
`UnitySim.ContactsBetween()`. These two calls are usually near each other inside
a FindPairs operation in a rigid body simulator. Here are some instances in
Anna:

-   [https://github.com/Dreaming381/Latios-Framework-Add-Ons/blob/v0.5.0/AddOns/Anna/Systems/FindCollisionsSystem.cs\#L159-L162](https://github.com/Dreaming381/Latios-Framework-Add-Ons/blob/v0.5.0/AddOns/Anna/Systems/FindCollisionsSystem.cs#L159-L162)
-   [https://github.com/Dreaming381/Latios-Framework-Add-Ons/blob/v0.5.0/AddOns/Anna/Systems/FindCollisionsSystem.cs\#L181-L184](https://github.com/Dreaming381/Latios-Framework-Add-Ons/blob/v0.5.0/AddOns/Anna/Systems/FindCollisionsSystem.cs#L181-L184)
-   [https://github.com/Dreaming381/Latios-Framework-Add-Ons/blob/v0.5.0/AddOns/Anna/Systems/FindCollisionsSystem.cs\#L202-L205](https://github.com/Dreaming381/Latios-Framework-Add-Ons/blob/v0.5.0/AddOns/Anna/Systems/FindCollisionsSystem.cs#L202-L205)

What makes this challenging to debug is that usually the results produced by
these bugs are only bad in context. In other simulations, or perhaps even at
other parts of the same simulation, such results are valid and expected. To
properly identify the issue, you’ll need to experiment with heuristics to catch
these bad results, and be ready to deal with false positives.

When debugging, it is best to put some if statements around those two methods to
try and catch something gone wrong. You can then use breakpoints, logging, and
`Debug.DrawLine()` to investigate. Printing logs as errors and enabling
pause-on-error in the console window can be especially helpful, as it will let
you see the debug lines you drew. You’ll probably want to draw the colliders in
question, as pausing happens at the end of the frame, and the bodies will have
moved a bit from where they were during collision detection.

While it isn’t always clear which values are signs that something has gone
wrong, here are a few hints.

First, if the distance of any contact is far more negative than the
`ColliderDistanceResult.distance`, that’s definitely a bug. The smallest contact
distance (with larger negative numbers being considered “smaller”) should be
near to the `ColliderDistanceResult.distance` within some floating-point error
tolerance. Though that tolerance may be higher than you would expect, closer to
`1e-4f` or so. Also watch out for infinities and NaNs.

Another sign might be that the `ColliderDistanceResult.distance` itself is
significantly negative or a very large positive value. This can be tricky to
identify though, because significant negative values are normal at first impact
of a moving rigid body, or when multiple rigid bodies get tangled up in a web of
collisions.

The third sign is when there is only a single contact point. This one is
especially tricky, because a single contact point is normal in a lot of
situations. You may want to draw out contacts and step through a few frames to
get a sense of where contact points end up various scenarios. But in general, if
there is an edge close to a face, or two faces close together, you should expect
multiple contact points.

One final tip is to assign a `FixedRateSimpleManager` to
`SimulationSystemGroup`. This will add some on-device determinism to your
simulation so that you can reproduce the same behavior entering play mode
multiple times.

## Reporting Bugs

Sometimes, it is good enough to provide a repro project that has a high
likelihood of producing unwanted behavior. However, this is very time-consuming
to debug, and decreases the chance the bug will be resolved.

The better option is to capture a hex string at the exact moment the
calculations go wrong, which can then be used to replay the bad evaluations with
a debugger. You can do this by calling `PhysicsDebug.LogDistanceBetween()` and
`PhysicsDebug.WriteToFile()`. This will write a sequence of hexadecimal
characters in plain-text to a file. You can distribute that file or
copy-and-paste that text in a bug report. Additionally, if one or both of the
colliders involved is a composite collider (compound, tri-mesh, or terrain), try
to specify the subcollider indices when you report the bug.

## Why Are These Issues Happening?

The short answer is that Psyshock and Unity Physics do collision detection
differently, and trying to bridge Psyshock’s collision model with Unity Physics
simulation algorithms has surfaced various problems. Eventually, `LatiosSim`
will solve this, as it will be designed from the ground-up with Psyshock’s
vision in mind. But that will take time to develop.

If that’s a good enough answer for you, you can stop here. But if you would like
a deeper understanding of the issues at play, and why they are so difficult,
read on.

Unity Physics uses GJK+EPA as the foundation for its collision detection. These
are algorithms that operate on the Minkowski Difference between two colliders,
and produce a simplex within that difference. Unity Physics uses GJK+EPA to
derive the contact normal, and uses that and the simplex to perform a search on
each collider for potential planes to do contact generation for. However, for
simple shapes, GJK+EPA is expensive, so instead Unity Physics performs an SAT
test, and then guesses at contact planes with the separating axis used as the
contact normal. After producing contacts this way, Unity Physics checks if any
of the contacts are near the SAT distance, and if not, it drops everything and
reverts back to GJK+EPA. Because of this, Unity Physics does narrow-phase
collision detection and contact generation all within a single method.

On the contrary, Psyshock tries to keep narrow-phase collision detection and
contact generation separate for improved modularity and customizability. But it
doesn’t make sense to pack the various collision hints Unity Physics uses inside
the `ColliderDistanceResult` for `ContactsBetween()` to use. It would break the
generality of `DistanceBetween()`. Instead, Psyshock reports closest feature
codes. A feature is a point, edge, or face on a convex collider. At the time of
writing, the feature codes are `internal` members of `ColliderDistanceResult`,
though they may be exposed indirectly through a utility API in the future. The
feature codes replace the GJK+EPA metadata for deriving the contact normal, and
this is where things sometimes go bad.

Sometimes, Psyshock mis-identifies the feature codes due to floating point
errors that weren’t caught correctly (more on this later). This results in
selecting an inappropriate contact normal, which then produces very strange
contact points. Part of this comes from Psyshock trying to emulate the contact
normals from Unity Physics SAT fast-path so that it can produce similar
behavior.

Another factor at play is that Psyshock uses very different data structures than
Unity Physics for storing colliders, and this comes into play for contact
generation. Unity Physics uses a half-edge data structure for convex shapes.
While the half-edge structure has some very cool properties for how everything
connects and can be iterated, it has some downsides in intuitiveness, memory
footprint, and SIMD. Unity Physics box colliders require nearly half a kilobyte
because of the half-edge structure, whereas Psyshock gets by with just 24 bytes.
But these differences have naturally resulted in bugs creeping into the
`ContactsBetween()` implementation.

## Why Not Use Unity Physics Algorithms for UnitySim Contacts?

At this point, that’s probably not a bad idea. And if that’s something you want,
feel free to try and add it to Psyshock. But `UnitySim` has some serious
drawbacks that likely make it not the future of Psyshock. Odds are one of these
drawbacks drew you to Psyshock in the first place.

An obvious issue is the lack of support for non-uniform scaling. 0-scale on one
axis gets especially dicey (and probably needs a lot more testing in Psyshock),
but the rest is surprisingly trivial to patch into the Unity Physics algorithms.

A bigger issue is that Unity Physics allows itself to generate nonsensical
contacts. For example, in Unity Physics, set up an upright capsule just above a
box floor and then rotate the capsule 45-degrees around the z-axis. Generate
contacts between the capsule and the floor, and you’ll see that while one of the
contacts is at the bottom surface of the capsule, another contact is in some
random-ish spot inside the capsule higher up. Unity Physics does a lot of
awkward number fudging around rounded shapes, despite encouraging their use a
lot more. The inconsistencies of this is a nightmare to deal with when you want
precise control and extensibility over your simulation. As of writing, Psyshock
does not support rounded boxes and convex meshes, so this bug only manifested
for capsule colliders. Psyshock attempts to rectify this issue by moving the
awkward contact point back to the capsule surface using raycasting.

Another issue comes from when you have a capsule that is locked upright, and
every frame gets pulled into a box floor from gravity. If you were to try and
rectify this by simply pushing the capsule out of the floor using GJK+EPA’s
collision normal and distance, something very strange would happen. The capsule
would very slowly drift along the box, even when it is supposed to be still.
(About 1 meter per 2.5 minutes.) This problem isn’t just floating-point error,
it specifically comes from EPA. GJK alone can only detect if there is a
collision, and only when there is not can it provide distance and info about the
closest points. If there is a collision, then it has to defer to EPA, which is
effectively a convex hull builder algorithm. Unity Physics’ convex hull builder
is quantized, meaning it produces quantization errors in the result. For shapes
with awkward geometry, this is a reasonable tradeoff. But for shapes with strong
axis-aligned properties, developers generally want very clean numbers.

While EPA is the more apparent issue, it is also often rectifiable with some SAT
post-analysis. The more catastrophic issue with GJK+EPA for Psyshock is GJK’s
unawareness of faces. GJK only reasons about vertices, and sometimes due to
floating point errors, it might return the closest points between two box
colliders as a point on the edge of one box, and a point on the face of the
other. When it comes to feature codes, face-edge and face-face are invalid
combinations. Physically, if such a closest combination exists, then there must
also be an equally close face-point or edge-edge pair. And the latter is a
desired contact point, so it is more useful. GJK’s reporting of face-edge pairs
breaks Psyshock’s ability to identify a correct contact normal.

One last area Psyshock improves upon is when contact generation generates more
than 32 contacts. Unity Physics discards the extra contacts as-is, which can
result in persistent wobbling for some convex mesh shapes. Psyshock reduces the
contacts using area optimization which avoids the issue. In the future, it is
desirable for Psyshock to provide user control over the number of final contacts
with this algorithm.

When you combine all of Unity Physics algorithms together, they all happen to
compensate for each other to produce a functional simulation, albeit a noisy
one. As soon as you try to replace parts with Psyshock that tries to be more
precise (and sometimes fails), things fall apart fast.

## What Does Psyshock Do? And What Does It Need To Do?

Currently, Psyshock’s solution to these various problems is messy and
inconsistent, and not at all solved yet.

Spheres and capsules use analytical brute-forced techniques for nearly
everything. There are two exceptions, where capsules use GJK+EPA against convex
meshes, and contact generation is based on Unity Physics with some special
Psyshock corrections. Some of these analytical solutions come from Unity
Physics, while others are custom-derived for Psyshock. They are usually quite
reliable.

Box vs box has had a wild history, using a very awkward SAT-ish custom solution
up until 0.10, when `UnitySim` exposed all of its bugs. From 0.10 through 0.12,
box vs box used Unity’s GJK+EPA implementation with no SAT optimization. This
worked okay-ish, until users started running into stability issues with
axis-aligned boxes. These were due to EPA’s quantization issue and GJK’s
face-edge reporting, therefore 0.13 would introduce a new full SAT box vs box
implementation. This worked great for axis-aligned boxes. But as it turns out,
there’s a lot of tricky floating-point edge cases for closest point and feature
code identification when things get slightly off-axis. For example, you might
have a small box on top of a big box. SAT might identify the separating axis is
between the vertices of the big box and the face of the small box. However, the
big box vertices are actually very far away from the small box. 0.13 introduced
a lot of regressions that have since been mostly worked out, as boxes are very
popular for some reason and get a lot of testing. SAT is often a lot faster than
GJK+EPA and more accurate. But it takes time to work out the floating-point
bugs, and using it for closest point determination on separated bodies is
academically underexplored.

Most likely, box vs triangle and triangle vs triangle will need to undergo the
same SAT treatment as more users start to use tri-mesh and terrain colliders.
But currently, they still use GJK+EPA. For convex colliders, GJK+EPA will have
to stay, because while SAT is very fast for simple geometry, it has quadratic
complexity compared to nlog n with GJK+EPA. So for those, new correction
algorithms need to be designed to compensate for GJK splitting faces and
possibly for EPA to use a localized SAT post-pass.

`UnitySim.ContactsBetween()` needs better logic selecting the contact normal,
and bugfixes for various collider pairs. The biggest challenge is that Unity
Physics’ algorithm for contact generation is not the easiest to understand.
Eventually, `LatiosSim` will derive a new algorithm from scratch that can be
better understood and more customizable.

However, Psyshock is just one module of many in the framework, and its focus has
been more tailored towards use cases that only need spatial queries, and not
rigid body simulation. These issues likely won’t be resolved in a timely manner.

## How To Help

Unity Physics has a team of developers with decades of experience. And they use
this to build lots of test cases, invest in debugging tools, and ultimately win
the game of whack-a-bug. For Psyshock to truly be competitive, it needs a
similar team composed of arbitrary members of the community. The team doesn’t
have to be as focused or committed or as stable, it just needs people willing to
take a little time each month, or even just once, to make things a little bit
better. Every bit helps, and with enough people, Psyshock can truly become the
solution for everyone.

So to help, the most important thing is to report bugs using the process
outlined above. But if you want to dive deeper, feel free to make an attempt at
fixing them.

The bug discovery and reporting process is inaccessible or impractical to some
developers and their projects, so another way to help would be to design and
build more enhanced tooling that can capture and replay simulations with the
ability to focus on specific collision pairs on specific frames.

Third, Psyshock could really benefit from a centralized project full of test
cases that could help identify bugs faster and reduce the risk of regressions.

And lastly, some of the next steps to improve upon GJK+EPA or convert algorithms
to SAT, or even make more direct ports of Unity Physics contact generation are
all fair game for anyone in the community to take on.

Thank you for reading to the end of this long ramble. Hopefully it provides
clarity on the situation.
