# Getting Started with Psyshock Physics – Part 5

In the previous part, we learned about PairStreams and working with data in cool
ways. But now, it is time to get back to physics. In this part, we’ll cover
motion and forces.

## Particle Physics

No, this is not quantum mechanics. We are going to talk about game particles.
The little points that move around based on physics rules. What makes something
a particle is that it doesn’t have rotation.

I’m going to assume you know the absolute basics of classical mechanics:

```csharp
changeInPosition = velocity     * time;
changeInVelocity = acceleration * time;
force            = mass         * acceleration;
```

In game dev, we typically use these like this:

```csharp
velocity += inverseMass * force * deltaTime;
position += velocity * deltaTime;
```

Why `inverseMass` instead of dividing mass? Well, that’s because game developers
like to make some physics things completely immovable by the rest of physics.
That can be done by giving it “infinite mass”, which can be represented with
`0f` `inverseMass`. Also, for the micro-optimizing enthusiasts, floating point
multiplication is typically 50% faster than floating point division in raw
throughput ignoring all other bottlenecks and latency-hiding tricks.

Now the expression force \* `deltaTime` is a common one in game dev and has a
special name: **impulse**. Sometimes, it is easier to reason about impulses
without knowing the factors that created them, so you may see impulses show up
in the Psyshock API more than forces.

Psyshock comes with a couple of utilities for calculating forces from other
properties. One of these is `Physics.DragForceFrom()`. This method has actually
two different forms. One form uses two drag coefficients named `k1` and `k2`.
Usually people just make up these numbers based on whatever feels right. For
something a little more realistic, there’s another variant that uses the “fluid”
(usually air) velocity, viscosity, and density. This version models the actual
friction of a spherical particle in these environments. Constants for air are
provided in `Physics.Constants`. Lastly, there’s `Physics.BouyancyForceFrom()`
which models the force of a fully submerged sphere.

## Rigid Body Positions

Particles are simple. Rigid bodies… not so much.

Rigid bodies often come in weird shapes, and they involve rotation, which has
all sorts of weird math shenanigans, especially in 3D space.

Let’s first talk about **center of mass**. This is the weighted average position
of all the atoms in a rigid body each multiplied with their atomic mass.
Calculating this generally requires intimate knowledge of the material
composition and distribution of an object. But if you assume the entire rigid
body collider is filled up fully with the same “stuff”, then there are simple
formulas to calculate the center of mass. Psyshock provides this via
`UnitySim.LocalCenterOfMassFrom()`.

*Note: Psyshock’s UnitySim APIs generally try to mimic Unity Physics formulas.
Unity Physics computes the center of mass of a mesh collider as simply the
center of the axis-aligned bounding box. Additionally, compound colliders sum
the masses even when the underlying colliders are overlapping, which can result
in non-uniform density of the overall shape. If you want a more accurate center
of mass, consider calculating it in a DCC app or other tool and storing it in a
component.*

The center of mass is special, because it represents the rigid body’s rotational
pivot point. That means if a rigid body is only spinning, the point of the rigid
body at the center of mass never moves. Some developers may try to author
entities so that the local-space center of mass of an entity is always zero.

Also, if your rigid body was not allowed to rotate, then the center of mass
would be the “particle position” and all the formulas for particle physics would
apply to such a rigid body as well.

## Rigid Body Rotations

Yup. This is the part that confuses everyone, myself included most days.

The key to making sense of rotation when working in Psyshock is to understand
the presence and complexity of something called the **inertia tensor**. That’s a
really scary term, but there’s some intuition behind it.

Imagine a tire. When the tire rolls the way it is meant to roll, it does not
take much force to keep it rolling. However, if the tire is lying flat, it takes
a lot more force to flip it over. This means that how much an object resists
rotation depends on which axis some force is trying to rotate it. And because
physics is weird, how far away the force is from the center of mass also makes a
difference.

Anyways, we represent this resistance in a 3x3 matrix, and that matrix is the
inertia tensor. Often times, we will compute the inertia tensor as if it had a
mass of 1.0 across the whole object, and then scale it dynamically at runtime.

Now, 3x3 matrices are really awkward. And as it turns out, there’s usually some
magic rotation matrix which you can multiply with an inertia tensor and get 0s
in every spot in the matrix except for the top left, middle, and bottom right
values. This is known as the **diagonal** of the matrix. This means that for any
shape, there’s some magic rotation that makes it so that the primary resistances
to rotation are all axis-aligned. Just like how the center of mass is the rigid
body’s pivot position, you can think of this magic rotation as the rigid body’s
pivot orientation. In Psyshock, `UnitySim.LocalInertiaTensorDiagonal` contains
the rotation called `tensorOrientation` and the 3 diagonal values in a `float3`
called `inertiaDiagonal`.

Something very interesting happens here. The diagonal values are the “perceived
rotational masses” for rotations of their given axis. This means if you applied
a rotational force (formally known as a **torque**) around the x-axis in the
local space of the inertia tensor diagonal, then `inertiaDiagonal.x` is the
mass-like resistance, such that:

```csharp
angularVelocity.x += inverseInertiaDiagonal.x * torqueAroundXInDiagonalSpace * deltaTime;
```

One tricky detail about the inertia tensor is that scaling and stretching it
changes its values in non-obvious ways. This is because when you scale the
shape, you are effectively spreading out the mass farther from the center of
mass, and this means the outside atoms need to be moved greater distances to
achieve the same rotation. Uniformly scaling an inertia tensor is cheap, though
not simply a scalar multiplication of the matrix. Stretching is more expensive,
because it changes the diagonal orientation. With Burst, you could still
probably get away with doing it for thousands of rigid bodies every frame, but
if you are trying to be optimal, understand that the cost is there. And
Psyshock’s APIs are designed with this in mind.

While you could compute the local inertia tensor diagonal and orientation
precisely with your own apps and tools, Psyshock provides
`UnitySim.LocalInertiaTensorFrom()` to compute them from uniform-density
colliders just like with center of mass which returns the
`LocalInertiaTensorDiagonal`. Note that in this method, you need to pass in the
`stretch` value up-front. And like center-of-mass, these methods make crude
approximations for triMesh colliders and self-overlapping regions of compound
colliders.

However, most physics simulations take place in world-space. Naturally, it is
desirable to have a world-space transform of a rigid body’s true pivot data. In
Psyshock, this is referred to as the **Inertial Pose World Transform**. This can
be obtained using the entity’s `TransformQvvs` transform, the local inertia
tensor diagonal, the center of mass, and the inverse mass. The method is
`UnitySim.ConvertToWorldMassInertia()`. It outputs the
`inertialPoseWorldTransform` as a `RigidTransform`. It also outputs a
`UnitySim.Mass` which contains the inverse mass as well as the `inverseInertia`,
which is the component-wise inverse of the inertia tensor diagonal.

Overall, the pattern for rotations is:

1.  Derive the `LocalInertiaTensorDiagonal` for a unit mass either using
    `UnitySim.LocalInertiaTensorFrom()` or your own derivation. Be ready to
    recalculate it if the stretch of the entity changes.
2.  Have the inverse mass and center of mass at the ready.
3.  Obtain the simulation `Mass` and `inertialPoseWorldTransform` from
    `UnitySim.ConvertToWorldMassInertia()` and store them. Be ready to
    recalculate them if the mass, scale, or inertia tensor changes.

## Rigid Body Motion

Now that we’ve broken down the rigid body’s mass and inertia tensor and have
world-space properties, we are ready to start influencing the rigid body with
forces. But before that, we have one more type we need to define, and that’s
`UnitySim.Velocity`. This contains both linear and angular parts. And wouldn’t
you know it, but the angular components represent the axis and magnitude of
rotation within the inertia tensor diagonal’s local space! If you have an
angular velocity of (2, 0, 0), that means it is rotating at two radians per
second around the diagonal’s local x-axis.

Psyshock offers two methods for applying impulses that hide away some of the
math. These methods immediately update the `Velocity`. The first is
`UnitySim.ApplyFieldImpulse()` which applies a uniform impulse across the entire
rigid body. Such an impulse will never alter angular velocity, and only “moves”
a rigid body. You provide the `Mass` and the impulse as a vector, where the
normalized impulse is the direction, while the magnitude is the amount of force
within the timestep. The second method is `UnitySim.ApplyImpulseAtPoint()` which
additionally requires the world-space position the impulse is applied at, and
the `inertialPoseWorldTransform`. This method may impact both linear and angular
velocity.

Now that we can update the `Velocity` with impulses, we need to apply the
velocity updates to the rigid body. We can do that with `UnitySim.Integrate()`.
This requires `deltaTime` along with damping factors. Typically in simulation, a
small amount of damping is always applied to prevent floating point errors from
causing things to “jitter until they explode”. The method updates both the
`inertialPoseWorldTransform` and the `Velocity`.

With the new `inertialPoseWorldTransform`, the final step is to map the
transform back to the entity’s transform. You will need the entity’s
`TransformQvvs` from the start of the simulation step (it probably hasn’t
changed yet). And you will need some way to map that to the inertial pose.

Usually, the more practical method is if you have the
`inertialPoseWorldTransform` from the start of the simulation step, you can use
that as the mapping. The method for this is
`UnitySim.ApplyInertialPoseWorldTransformDeltaToWorldTransform()`.
Alternatively, if you don’t have the previous `inertialPoseWorldTransform` but
do have the local-space inertial pose, you can use
`UnitySim.ApplyWorldTransformFromInertialPoses()`.

## Rigid Body in a Single Job

That was a lot of information, but we can summarize the entire API for a
simulation step with forces and motion updates in a single `IComponentData` and
`IJobEntity`. Here it is:

```csharp
struct RigidBodyData : IComponentData
{
    public UnitySim.Velocity velocity;
    public UnitySim.Mass     mass;
    public RigidTransform    inertialPoseWorldTransform;

    public static RigidBodyData CreateInBaking(TransformQvvs worldTransform, Collider collider, float inverseMass, float3 staticStretch)
    {
        var localInertiaTensorDiagonal = UnitySim.LocalInertiaTensorFrom(in collider, staticStretch);
        var centerOfMass               = UnitySim.LocalCenterOfMassFrom(in collider);
        UnitySim.ConvertToWorldMassInertia(worldTransform, localInertiaTensorDiagonal, centerOfMass, inverseMass, out var mass, out var inertialPoseWorldTransform);
        return new RigidBodyData
        {
            velocity                   = default,
            mass                       = mass,
            inertialPoseWorldTransform = inertialPoseWorldTransform
        };
    }
}

partial struct SimJob : IJobEntity
{
    public float deltaTime;

    public void Execute(TransformAspect transform, ref RigidBodyData rigidBody)
    {
        // Apply gravity
        rigidBody.velocity.linear.y -= 9.81f * deltaTime;

        // Apply linear force of 5 N upwards
        UnitySim.ApplyFieldImpulse(ref rigidBody.velocity, in rigidBody.mass, new float3(0f, 5f, 0f) * deltaTime);

        // Apply a force of 5 N upwards offset by one meter from the entity's center
        UnitySim.ApplyImpulseAtWorldPoint(ref rigidBody.velocity,
                                          in rigidBody.mass,
                                          rigidBody.inertialPoseWorldTransform,
                                          transform.worldPosition + transform.rightDirection,
                                          new float3(0f, 5f, 0f) * deltaTime);

        // Save a copy of the old inertialPoseWorldTransform before modifying it.
        var oldInertialPoseWorldTransform = rigidBody.inertialPoseWorldTransform;

        // Update motion with 0.01f damping factors for linear and angular.
        UnitySim.Integrate(ref rigidBody.inertialPoseWorldTransform, ref rigidBody.velocity, 0.01f, 0.01f, deltaTime);

        // Apply back to entity's transform
        transform.worldTransform = UnitySim.ApplyInertialPoseWorldTransformDeltaToWorldTransform(transform.worldTransform,
                                                                                                 oldInertialPoseWorldTransform,
                                                                                                 rigidBody.inertialPoseWorldTransform);
    }
}
```

## Up Next

There’s still plenty more to explore in the Psyshock API. We haven’t even
covered contacts or other simulation APIs. But until I get around to expanding
this series, feel free to explore them on your own.

Psyshock can be intimidating. No question is too stupid and you are encouraged
to reach out to the community on Discord if you want help or would like to get
more performance out of your use case.
