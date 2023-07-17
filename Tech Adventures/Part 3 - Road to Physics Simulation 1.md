# Tech Adventures: Road to Physics Simulation Part 1

Somehow through the years of my game development endeavors, I’ve never really
made use of the simulation capabilities of physics engine. I typically only ever
rely on triggers and otherwise fake physics-based motion with custom code, akin
to how kinematic character controllers work.

Even though Latios Framework 0.8 is compatible with Unity Physics, Unity physics
has lots of issues that more and more people are starting to bring their
complaints to me about. Unity Physics is too black-boxy and has issues with
stacking and contacts performance and customization of joints and lack of
transform hierarchy support and Havok is behind a \$1500 Pro license paywall and
you get the idea. Psyshock’s “build-your-own-physics” architecture is very
appealing. At the very least, learning simulation will let me use physics to
animate hair. I don’t know why, but seeing hair move in response to head
movement reminds me that there is a consciousness inside the head that
deliberately chose to create the movement that the hair reacted to, perhaps a
representation of the diversity of concerns consciousness juggles
simultaneously, life in its truest form…

Alright. I’m rambling now. Let’s move on.

On my quest to learn physics, I’ve noticed there is a lot of terminology and not
really a good free resource as a starting place for 3D. A lot of people just
suggest investigating the Box2D subforums. You go there, and you hear things
about broad-phase, narrow phase, contact generation, clipping, integrators,
solvers, impulses, and a whole bunch of other terms that don’t really mean
anything if you aren’t already versed in the domain.

The only book I have been able to get my hands on in this domain is Ian
Millington’s *Game Physics Engine Development*. I’ve only read part way through,
but I can safely say this book will **not** teach you how to build a commercial
grade physics engine. The techniques used are a bit outdated and outclassed, and
the example code is architecturally very misguided, which makes the code itself
more obfuscated than necessary. However, what this book does well from what I
have read so far (yeah, I haven’t finished yet) is explain the domain language
of physics engines. It also provides a little bit of history of physics engines
(being older techniques), so when you do go look at good resources like the GDC
talks, not only will you be able to understand them, you’ll also be able to
understand why those more modern techniques are better.

However, since Psyshock’s goal is to provide a toolset for you to build your own
physics engine, you’ll need to understand all these pieces as well. That’s the
main reason I am documenting this adventure, to help you understand the
different pieces and tradeoffs so that when this tech is ready, you can actually
take advantage of it. And maybe you can even help with the development process
along the way!

## Physics Engines are Fantasies

In the real world, nothing is allowed to break the laws of physics. While that
may be a fun clickbait title, it is never actually true. There’s always some
rule to the physics of our world which drives the behavior we see, whether we
understand it or not. That’s true regardless of which world I wake up in.

However, it is not true for physics engines. Physics engines break the laws of
physics all the time! They kinda have to.

First off, we have to take jumps in time. While the several millisecond
timesteps might seem small in real-time, in the same way what we perceive as
instantaneous interactions are actually a complex sequence events. Take a ball
bouncing on a pavement. It might seem like the time the ball touches the
concrete is instantaneous. But in reality, there’s a whole sequence of
interactions involving the elasticity of the ball’s shell, the pressure of the
air inside the ball, and the heat dispersion that reverberates throughout.
Because of our time steps, we don’t get to simulate that accurately. And while
our simulation gets closer and closer to accurate the smaller we make our
timesteps, it also becomes more computationally expensive until we reach a point
where we can no longer achieve real-time simulation. In addition, smaller
timesteps mean smaller numbers. And with smaller numbers, our calculations need
to be more precise, which requires more memory to store.

Instead of trying to prevent the laws of physics from breaking, we just let them
break, and then try to correct results as closely as we can.

This leads us to the first big difference between physics engines, which is the
simulation model for performing these corrections. There are two main
techniques: velocity-based and position-based.

Velocity-based techniques (I believe sometimes also referred to as “impulse
solvers”) try to resolve breakages of the laws of physics by correcting the
velocities of the dynamic objects. This approach is pretty popular nowadays and
is used by Unity Physics and Havok among others.

The other approach is position-based, or rather position-based dynamics. This
approach historically had accuracy issues relative to time, but more recently
those issues have been rectified with the new extended variants. There’s not
nearly as many physics engines using it, but it has been seeing a rise in
popularity with ECS ecosystems as of late.

The biggest difference I can see between them is that velocity-based methods are
more prone to translational errors whereas position-based methods are prone to
rotational errors. I suspect these differences will become clearer later on, but
if you’d like to read a far more detailed analysis, this paper has a very good
breakdown: <http://wscg.zcu.cz/wscg2023/papers/short/G05-full.pdf>

## To State or Not to State

Another way to classify physics engines is by whether they are stateful, or
stateless. However, these terms are misleading because it isn’t about whether
there is a presence or lack of state, but rather which pieces of code keep track
of it.

In a stateless engine, the user of the engine owns the state. In the case of
Unity Physics, the ECS components own the state. In Havok, the simulation
backend remembers things about each Entity that the rest of the ECS can’t
access, and those memories affect the simulation. That’s why it is stateful.

The stateful components in the Physics samples are also misleading. They don’t
make Unity Physics stateful. They just happen to be state that is tricky to
snapshot and rollback in NetCode in a way that doesn’t cause problems.

By this logic, the fact that Unity Physics doesn’t have features like sleeping
rigidbodies, tall stable stacks, and contact welding has nothing to do with the
fact that it is stateless. Even though I don’t fully understand physics engines,
I do know that physics engines are governed by the rules of general computing.
And state management falls under the rules of general computing.

So why does Unity Physics not have those features? I would speculate part of it
is due to technical debt from the architecture. And some of it may be capitalism
chess.

## What is ECS Physics?

Not Unity Physics.

Unless you are familiar with other ECS ecosystems, this may come as a surprise.
But there’s a big difference between being an ECS physics engine and being an
ECS-compatible physics engine. That difference is whether the physics engine is
embedded, or independent.

But to make this as blunt as possible, this all has to do with where the physics
engine stores its temporary calculations. Unity Physics copies all of the ECS
state into a `PhysicsWorld`. Then it does the calculations there. And finally
when it is done, it writes the data back to ECS. You could replace the gather
and writeback operations with a different data architecture, and the simulation
would still function the same (mostly).

On the contrary, a true ECS physics engine will store its temporary state
directly inside ECS components. Things like contact points and constraint
details would be in `IComponentData` and dynamic buffers. On the upside, it
makes the physics much more customizable. A potential downside of this technique
is that ECS isn’t spatially-structured. However, Psyshock solves this problem
with `CollisionLayers`. Sure, you might argue that `CollisionLayers` copy state,
but FindPairs lets you look up components and buffers in parallel.

In fact, I’ve considered removing most of what is in the `CollisionLayer` and
requiring the user to look them up instead. But as it turns out, there’s a good
reason to keep things the way they are. One of the pieces is the Entity, which
is necessary for lookup. The transform was originally added because it used to
be a lot more expensive to extract from ECS. However, nowadays it provides
performance boosts for triggers (the much more common use case of FindPairs) and
acts as a reminder that the AABBs used for FindPairs don’t dynamically update.
However, you can always bypass the stored transform and use the most up-to-date
transform instead. We’ll get to reasons we may want to do this in a bit. Lastly,
there’s the collider itself, which is convenient to store in the
`CollisionLayer` because there are often cases where we dynamically generate the
collider each frame and then we don’t have to store it in ECS components. It
also lets us have colliders as fields inside other components and select which
gets used for different collision layers. It will be even more powerful once
layer compounds are supported, which will allow for dynamic skinned mesh
colliders and such.

All that is to say that Psyshock will facilitate the creation of both embedded
and independent physics engines. There are pros and cons to both approaches, and
the decision will largely depend on your needs.

## Anatomy of a Physics Simulation

Now that we’ve gotten all that background out of the way, it is time to break
down the various steps we need in order to have a working simulation. Along the
way, we’ll assess what Psyshock already has and what is still needed.

### The Integrator

The integrator has to do with integration. Not in the software and business
sense, but in the calculus sense. *Groaning anticipated.*

Fortunately, we don’t actually have to calculate integrals. Instead, we use
Riemann Sums instead, which might sound just as scary, but is actually so simple
that you’ve probably done it without even realizing it.

Velocity is change in position over change in time. Therefore, position is the
integral of velocity. But if you just want to calculate the position at each
timestep given a velocity function, all you need to do is evaluate the velocity
function at each timestep, multiply the result by `deltaTime`, and add it to the
position.

```csharp
position += EvaluateVelocity(currentTime) * deltaTime;
```

That’s effectively what the integrator does. Similarly, acceleration is the
change in velocity over change in time. And of course you remember *F = ma* from
school, right?

Psyshock has integrator utilities for particles, such as computing buoyancy and
drag forces. With them, I was able to make a realistic fireworks simulation.
However, Psyshock still needs integrators that deal with spinning objects.
That’s an area of future investigation. Motors also get implemented here.

### Collision Detection

Moving bodies around in free space is fun, but gamers are way more entertained
by things smashing into each other. Collision detection is all about figuring
out when and where that happens.

Efficient collision detection is broken up into 3 phases, broad-phase,
mid-phase, and narrow phase.

The broad-phase tries to rule out colliders that are definitely not colliding
with as few calculations as possible. Psyshock has a very optimized broad phase
in the form of FindPairs.

To understand the mid-phase, you have to understand something about colliders.
All collider shapes can be classified into two types, primitives and composites.
Primitives are shapes that are fully convex. They may also have have zero
volume, so a point, a line segment, and a triangle would all be primitives.
Spheres, capsules, boxes, and convex colliders in Psyshock are all primitives.
Meanwhile, composite colliders are composed of multiple primitives and may have
concavities. Compound, TriMesh, and Terrain colliders would all be composites
(Disclaimer: terrains aren’t implemented in Psyshock yet).

The mid-phase only applies to composites. It’s goal is to find potential
collisions between primitives within the composites as cheaply as possible,
similarly to the broad-phase. The big difference compared to the broad-phase is
that the mid-phase structures are in the local space of the collider, and so
transform space conversions need to be handled.

Psyshock doesn’t really expose the mid-phase, but instead its query methods just
provide the results of the most-interesting narrow-phase result across all the
primitives detected by the mid-phase. Psyshock’s mid-phase is also not nearly as
optimized. They haven’t been a bottleneck yet in any of my projects. But if they
are a problem in yours, let me know and I will make them faster.

The narrow phase is the most interesting. This is the phase the calculates the
specific details of intersection between shapes. `DistanceBetween` queries
return the closest points, the surface normals at those points, and the distance
between them (negative if intersecting). Meanwhile raycasts and collider casts
provide hit points, surface normals at those hit points, and the distance the
ray or collider traveled before the hit was registered.

Psyshock has a unique narrow phase method for each combination of colliders for
each query type. Some of these methods are optimized, while others aren’t. Just
like the mid-phase, if you notice performance problems due to the narrow phase
computations, please report the issue.

Overall, Psyshock’s collision detection algorithms are pretty comprehensive.
They provide everything you need for gameplay spatial queries and triggers. If
there’s more work to be done here, we’ll only know about it due to the later
stages.

### Contact Generation

The next stage is contact generation. Not only does the physics solver need to
know that things collide, but it needs to know the various points of contact.

One approach to contact generation is to add the closest pair of points between
two objects in local space per frame, and then each subsequent frame check if
those points are still intersecting. This is often referred to as incremental
contact generation. It is the approach used in Ian Millington’s book, and has
been used in commercial games. But the big downside of this technique is that it
takes several frames to find all the contacts, and that can cause objects to
rotate themselves through walls and floors.

Incremental contact generation requires no additional effort other than tracking
and storage. Psyshock’s collision queries provide all the relevant queries to
perform this.

The approach Unity uses however is they generate a multi-contact manifold. That
means they find all the interesting contact points each frame. For any pair of
primitives, all the contacts get collected into a *manifold*. In order to
compute the contacts, Unity analyzes the features of the closest points. A
feature being a vertex, edge, or plane of the shape ignoring any smooth radius
around the shape. Psyshock does not expose the features yet. Fortunately, for
spheres, capsules, boxes, and triangles, a feature can be represented in a
single byte. Unfortunately, convex colliders don’t work that way. They can have
up to 255 vertices, and even more edges. I’m hoping 2 bytes will be sufficient,
but I can always increase it to 4 to be safe, which would mean 8 extra bytes in
`DistanceBetweenResult`. Honestly, that’s not bad.

The other thing Unity does is compute a manifold for each primitive pair,
including for each primitive pair combination inside composite colliders. This
means that I’ll probably need to expose the mid-phase. A `DistanceBetweenAll`
method with a processor for each primitive result should do. That will also help
with character controllers.

There’s more we could investigate here too, such as things like speculative
contacts and contact welding. And different solvers prefer contacts at different
locations. But we’ll dive deeper into such topics at a later time.

### Constraint Solvers

The last piece of the puzzle (I think) is the constraint solvers. These I’m the
least familiar with. But essentially, every contact and every joint generate a
constraint. It is then up to the solver to try and satisfy these constraints by
changing physics properties such as positions or velocities. This largely
depends on the model used, which we discussed earlier.

But there is another factor at play here. There’s this concept of “convergence”.
You see, constraints counteract each other. Just iterating through all the
constraints once will leave some constraints unsatisfied because other
constraints undid their effects. If you solve the constraints over and over
again multiple times, the simulation eventually converges to a more accurate
result. The goal of every physics engine is to “converge fast”.

One approach is to solve each constraint one-by-one, where the results of the
solved constraint are used when solving the next constraint. This is referred to
as Gauss-Seidel. Because it is modifying everything it touches, the algorithm
doesn’t work well in parallel without special techniques for resolving race
conditions like islanding or graph coloring. However, Gauss-Seidel is known for
good convergence speed. Fortunately, FindPairs has thread-safety between pairs
of colliders in parallel. So at least for contacts, a parallel Gauss-Seidel
implementation is fairly trivial to implement. I’ve actually done a drastically
oversimplified version of it in 2D.

The approach Unity Physics uses is a Jacobian solver. This solver tries to solve
all the constraints in parallel without considering the output of each
constraint solve as input into the next. While it is parallel and makes better
use of worker threads, it can converge more slowly.

There’s also a hybrid of the two, known as block solvers, which solve a small
number of constraints simultaneously. A good candidate for this is multiple
contacts between the same pair of colliders. Doing this can greatly improve
rotational stability of Gauss-Seidel solvers.

There’s lots to experiment with here.

## What’s Next?

The next big action item is contact generation. That means exposing the
mid-phase, making `DistanceBetween` provide feature details, and then performing
the multi-contact generation algorithms.

I will likely find opportunities to improve the integrators when I work on the
constraint solvers, as those will help me understand what state needs to
persist.

Do you think writing this article was worth 30% of my weekend dev time? Or
should I spend that time actually delivering these features? Let me know in the
Latios Framework Discord. And regardless,

Thanks for reading!
