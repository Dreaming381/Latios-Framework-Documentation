# UnitySim

The static class UnitySim provides many of the underlying types, algorithms, and
utilities from Unity Physics for creating a rigid body simulation. With these
APIs, you can recreate the essential feel of Unity Physics, but without the
overhead of unused features and customized to your needs.

While you don’t need to know all the advanced mathematical equations that a
physics simulation relies upon, you do need to understand some concepts behind a
physics engine in order to assembly a proper simulation. These are not the
easiest concepts to grasp, but they can be very rewarding once mastered.

As a prerequisite, you should familiarize yourself with the solver stages of
Unity Physics, detailed
[here](https://docs.unity3d.com/Packages/com.unity.physics@1.2/manual/concepts-simulation.html),
[here](https://docs.unity3d.com/Packages/com.unity.physics@1.2/manual/physics-pipeline.html),
and
[here](https://docs.unity3d.com/Packages/com.unity.physics@1.2/manual/simulation-modification.html).

## Terminology

We’re going to assume you know the basic physics terms that are typically taught
in school. Concepts like velocity, acceleration, forces, gravity, and mass.
These will not be discussed here. Familiarity with other common physics engine
terms like colliders and rigid bodies (and kinematic bodies) is also assumed.

The first topic is the **center of mass**. You may know this as the “balance
point” of an object. But more accurately, this is the point around which a rigid
body rotates. The key distinction here is that this point may not be the
entity’s position. This means that if a rigid body is naturally rotating
in-place, the entity’s transform position may be traversing in a circle.

Next, there is **inverse mass**. For a mass **m**, inverse mass is simply **1 /
m**. An infinite mass would have zero inverse mass, which is typically how
kinematic bodies are represented. Inverse mass shows up in the physics solver
far more often than normal mass, and it is often more optimal to store inverse
mass instead of mass.

The **inertia tensor** is perhaps where most people start to get confused. This
is a 3x3 matrix which describes the resistance to rotation to a force from any
direction. Different shapes have different resistances to rotation from
different directions of forces. For example, a tire can usually roll easily.
However, if the tire is laying flat on its side, then flipping it over can
require a great deal of effort. So much effort that this is actually a workout
exercise. The inertia tensor describes this. Often, physics engines may use a
**normalized inertia tensor** which is the inertia tensor matrix before being
multiplied by the average mass of the object, allowing the density (average
mass) to be altered on-the-fly.

Matrices are cool and all, but there’s an even more efficient way to represent
an inertia tensor. If you were to rotate the base shape around, the inertia
tensor would change dynamically. Eventually, you may find an orientation where
the matrix looks like this:

```math
\begin{bmatrix}a & 0 & 0\\0 & b & 0\\0 & 0 & c\end{bmatrix}
```

In this situation, a, b, and c are referred to as the **inertia tensor
diagonal** and the orientation that produced this is the **inertia tensor
orientation**. The orientation changes if the object is stretched. We can also
derive a **normalized inertia tensor diagonal** from a **normalized inertia
tensor**. This is NOT a vector of unit length 1, but rather a value that when
multiplied by the total mass of the rigid body produces the real inertia tensor
diagonal. Similar to inverse mass, we also typically work with an **inverse
inertia tensor diagonal**, which is simply the vector (1/a, 1/b, 1/c).

Imagine that for a rigid body entity, we had a child entity that was positioned
at the center of mass, and with an orientation set to the inertia tensor
orientation. If we did all our math relative to this child entity, the
transformation calculations become much simpler. We can move and rotate this
child entity around in world-space, and then compute the transform of the parent
using the child entity’s local transform, which we know to be derived from the
center of mass and the inertia tensor orientation. In Psyshock, we refer to this
local transform as the **inertial pose**. And when we describe this child
entity’s transform in world-space, we refer to it as the **inertial pose world
transform**. An important thing to keep in mind is that the rigid body’s
collider is NEVER stored in this virtual child entity. It’s properties are
always relative to the real entity.

For rigid bodies, velocity is broken up into two parts. There is **linear
velocity**, which represents translational movement and is typically the
velocity you think of. It is a vector value measured in signed meters per second
per axis in world-space. Then there’s **angular velocity**, which represents how
the rigid body is rotating. This one is measured in radians per second around
the principal axes relative to the inertia tensor orientation.

Forces can be applied either solely to affect the linear velocity across the
entirety of the object (how gravity works) or it can be applied at a point in
which case it will affect both the linear and angular velocities. However, since
forces apply changes to velocity over time, it is typically simpler to reason
about **impulses** instead. An impulse is simply the force multiplied by
`deltaTime`.

Unity Physics uses **specular contacts**. That is, instead of moving the rigid
bodies and then finding out what collided, it instead tries to predict what will
collide before moving them, so that it can integrate the result of the collision
into its movement. Naturally, this means that collision detection needs to
expand the AABBs and max distance values to find these collisions. This is
referred to as **motion expansion**. And one of the inputs of motion expansion
is the **angular expansion**, which is a rate at which the AABB expands per
radian of rotation.

Physics engines often calculate interactions through the use of **constraints**.
Constraints are rules which the physics engine tries to follow regarding the
bodies. Constraints can define how objects bounce off each other, or stick
together, or rotate only one axis. Unity Physics uses impulse-based dynamics,
which means it assumes necessary forces exist to enforce the constraints, and so
it applies impulses to the objects to ensure the constraints are satisfied.
Constraints are always defined between two rigid bodies. A pair of bodies may
have more than one constraint. A **constraint solver** loops over all
constraints, and applies impulses directly to the velocity values to ensure the
constraint will be satisfied. However, constraints will often undo the work of
previous constraints, which is why a constraint solver typically performs
multiple **solver iterations** where each iteration loops over all the
constraints. How many iterations are required to reach a stable solution is
known as the **convergence**, where fewer iterations required is better. The
Unity Physics solver has very good convergence, often only requiring 4
iterations for good results.

An optimization for a constraint solver would be to pre-calculate specific
values for the constraint before the solver loop, making those calculations only
happen once, and then storing them. Unity refers to these stored intermediate
structures as **Jacobians**. I will not comment here on the name choice, but for
clarity of mapping `UnitySim` to Unity Physics, Psyshock uses the same term.

When two objects collide, in order to determine how they bounce off each other
or rest against each other, we need to know multiple **contact points** between
the objects. Because Unity Physics uses specular contacts, a contact point is
actually a pair of points, one on each body’s surface. They also have a signed
distance between them along a **contact normal**, which is a normal vector that
points outwards from the second body in the pair.

We also need to know the amount of bounciness between the objects, which is the
**coefficient of restitution**, and we need to know how much resistance there is
between the surfaces of the rigid bodies sliding across each other, which is the
**coefficient of friction**. We can also account for the surface of a rigid body
moving independently, creating a **surface velocity**. An example of this would
be the moving part of treadmill.

For non-contact constraints, the “rigidity” of the constraint is defined by a
**damped spring** parameterization. Stiffer springs will have a higher **spring
frequency**, and springs that generally resist motion at all will also have a
high **damping ratio**. These parameters can be normalized for a constraint
solver, being transformed into values **damping** and **tau**.

Unity Physics is stateless, which means it has no memory of how it solved
constraints in previous frames and therefore due to floating point errors can
struggle to keep objects at rest. Minor cheats can be applied to the constraint
solver to correct this. And this process is referred to as **motion
stabilization**. Motion stabilizers require keeping count of **significant other
bodies** in a simulation frame.

Finally, once the constraint solver is finished and the velocity values are
updated, Unity Physics can move the rigid body entities. This process is
referred to as **integration**.

## Simulation Sequence

There are many ways to assemble a simulation, and many decisions to make to
tailor the simulation for your needs. This section will briefly describe the
general steps required, and some tips for implementation.

The first step is to build out the `CollisionLayers` for both static and dynamic
bodies. You can split these up into separate layers however you like. The key
detail is that for dynamic bodies, you should be using the
`Physics.BuildCollisionLayer()` which accepts `overrideAabbs`. These should be
the motion-expanded `Aabb` instances you can generate from
`UnitySim.MotionExpansion`. You will need to hold onto
`UnitySim.MotionExpansion` for inside FindPairs too. While building your
`CollisionLayers`, this can also be a good time to reset a `MotionStabilizer`
and the count of significant other bodies. And if necessary, you may also want
to rederive the inertial pose world transform. You can also apply gravity here
if you haven’t done it in an early state. Static body `CollisionLayers` can be
built using `EntityQueries`.

The next step is performing collision detection, where you use FindPairs. In
Unity Physics, this is separated into body pairs, contacts, and jacobians. But
in Psyshock, you can do all three steps in one go. Use the `MotionExpansion` to
compute the required `maxDistance` for `Physics.DistanceBetweenAll()`. And for
each `ColliderDistanceResult`, call `UnitySim.ContactsBetween()` and use that to
generate the contact jacobian via `UnitySim.BuildJacobian()`. You can store the
results of these calculations into a `PairStream`.

While collision detection is running, you may wish to build the constraint
jacobians for your non-contact constraints, such as joints. You can write these
to separate `PairStream` instances. You will need the `CollisionLayer` bucket
indices to do this. You can get that from a
`CollisionLayerBucketIndexCalculator`. Make sure to use the `MotionExpansion`
`Aabbs` when you do this.

Once you have assembled all your `PairStream` instances, it is a good idea to
concatenate them into a single instance to reduce the number of jobs you will
have to schedule in the constraint solver. However, this isn’t critical if not
using parallel jobs.

For each solver iteration, you will want to use `Physics.ForEachPair()` to solve
your constraints using `UnitySim.SolveJacobian()`. For static bodies, you can
provide default mass, velocity, and stabilization values. After each
`ForEachPair()`, you should run a stabilization update pass over all your
bodies, calling `UnitySim`. `UpdateStabilizationAfterSolverIteration()`. The
significant body count can be obtained either in FindPairs or the first
iteration of ForEachPair. If the former, be careful about using
`ScheduleParallelUnsafe()`. `Interlocked.Increment()` is viable for this use
case

Lastly, you are ready to perform integration by calling `UnitySim.Integrate()`.
After that, there are two different methods in `UnitySim` to apply the changes
to the inertial pose world transform back to your rigid body entity’s transform.
Pick the method that better fits the data you have available.

If you followed all these steps, you should have a functional rigid body
simulation. Now you can go in and apply all your customizations and
optimizations to tailor it to your needs!

## That’s All?

Psyshock’s inside-out approach to physics engines is a novel concept, and it
will take time and experimentation to build out best practices, examples, and
other learning resources. Any physics enthusiast is welcome to help out with
creating these resources and advancing the technology.
