# Tech Adventures: Road to Physics Simulation Part 8

We now have rigid bodies colliding with each other. And we have the ability to
apply forces to them and keep them stabilized. Psyshock is totally ready to be
used with your awesome characters in Kinemation…

What do any of the features have to do with characters exactly?

Nothing.

We should fix that.

There’s lots of physics that can be applied to characters, such as ragdolls,
dynamic bones, or softbody simulations. And in more advanced use cases physics
can be used to add realism to procedural motion. It is time to start building
this functionality, and we’ll begin with the essentials for rigid bodies: joints
and motors.

## What Even Are Joints and Motors?

We’re very fortunate to have Unity Physics as a reference implementation. Even
though I don’t often agree with their architecture, there’s a lot to be learned
from their algorithms.

A natural starting point would be to look at authoring and get a feel for the
designer parameters, and then trace things through the system. But don’t do
that. The authoring and baking code is a bit of a nightmare to decipher.
Instead, we can focus on the runtime components, and use both the authoring and
runtime inspectors to match things up.

Unity Physics defines three component types in PhysicsJointComponents.cs:
`PhysicsConstrainedBodyPair`, `PhysicsJoint`, and `PhysicsJointCompanion`.

### PhysicsConstrainedBodyPair

`PhysicsConstrainedBodyPair` specifies the two entities in the joint, and an
integer (which is really a bool) specifying whether collision should be enabled.
Unity Physics has some special higher-level rules for how joints and collisions
occur, and will dynamically disable collision pairs based on these rules. This
is higher-level architectural logic than what Psyshock aims to provide, so we
can completely ignore that part. That just leaves the two entities. Simple
enough.

What Unity’s XML docs won’t tell you is that `EntityB` is allowed to be
`Entity.Null`. In that case, it will be replaced with a virtual static physics
body located at the origin. The Limit DOF Joint authoring component will
generate this kind of configuration, which means in Unity Physics, DOF
constraints are implemented by a joint between the entity and the origin. It is
a little bit of a strange hack, but in practice isn’t all that different from us
providing default mass and velocity values for static bodies in the contact
solver. Improving this would be a topic for Optimization Adventures.

### PhysicsJoint

`PhysicsJoint` has a number of fields. The first two are `BodyFrame` instances.
We’ve never seen these before, and they can be a bit hard to comprehend if you
miss some of the XML docs, since the info is spread across the type, the
properties, and the fields within. But essentially, the `BodyFrame` is simply a
local-space transform within the body that specifies where the joint is located
and a general orientation for matching. For example, in a fixed joint, body B
would be positioned and oriented such that its `BodyFrame` transformed into
world-space would be identical to body A’s `BodyFrame` transformed into
world-space. Unity uses the position, the forward axis, and the up axis to
describe the body. However, instead of two axes, we could use a quaternion to
express the same thing and save a little memory.

Next, there’s a version property, which is a Havok caching thing and nothing we
care about.

After that, there’s a JointType enum. This lists out all the high-level joint
and motor types. However, this property is a “hint”. Nothing in the actual
simulation engine uses it. Which means Unity Physics doesn’t care about what
“joints” are at all. They are simply a composite abstract concept over something
else, constraints.

`PhysicsJoint` contains a fixed list of max capacity 3 of the `Constraint` type.
Each `Constraint` has an `enum` specifying its type. The types are as follows:

-   Linear
-   Angular,
-   RotationMotor,
-   AngularVelocityMotor,
-   PositionMotor,
-   LinearVelocityMotor

These are what Unity Physics actually cares about. The reason `PhysicsJoint` has
a fixed capacity of 3 instances is because most joints and motors require at
most one Linear, one Angular, and one of the motor constraints.

This then begs the question: What are constraints?

### Constraints Explained

Listen up! Because this might just be the most important thing you need to
understand if you want to keep up with all future developments of Psyshock!

Constraints are rules. They are not formulas and mathematical expressions;
rather they specify things that should be true.

A collision constraint might specify that two objects should not be
intersecting. A positional constraint might specify that two local positions on
two bodies should share the same world-space position.

The part of the Physics Engine that enforces these rules to be true is called
the **constraint solver**. In our demo physics engine in Free Parking, this is
what we are using `Physics.ForEachPair()` to do. How a physics engine handles
constraints is dependent on the type of physics engine. A position-based
dynamics engine will enforce the constraints by setting the positions and
rotations directly, and then back-calculating the dynamics that were required to
satisfy them. A mass-spring engine will convert every constraint into a spring
force and integrate all the forces.

Unity Physics uses impulse dynamics, which is the most popular type of physics
engine. Here we assume that there are arbitrary forces present that ensure the
rules are satisfied, and that these forces apply impulses each step. Therefore,
if we compute the minimum impulses that ensure the rules are satisfied after
integration, then we’ve effectively incorporated those forces without directly
calculating them. With this idea in mind, we can be a little more articulate in
how we define our rules.

In a collision constraint, we want the velocity to be such that after
integration, the bodies are not intersecting. In a positional constraint, we
want the linear and angular velocities to be such that after integration,
specific points on each body share an identical world-space position. In a magic
energy siphon constraint, we want the velocity to be such that the kinetic and
potential energy of the object at the end of the timestep has decreased by a
fixed rate relative to the beginning of the timestep.

As you can see, we can come up with quite complicated and creative constraints
if we desire. If we base the rules on reality, we get realistic results. And if
we base the rules on fantasy, we get fantasy-like results. Unity Physics defines
a collection of very general-purpose constraints. But the beauty of Psyshock is
that you are also able to define your own. As we explore Unity’s constraints,
we’ll hopefully pick up on some tidbits that will allow us to design our own
constraints in the future.

### PhysicsJointCompanion

The last component type Unity Physics defines is the `PhysicsJointCompanion`,
which is an `IBufferElementData`. This type isn’t used in the solver at all, and
seems to be provided for convenience for some complex joint types that require
more than 3 constraints to stabilize. We’ll ignore it for now.

## Here’s the Plan

With contact constraints, we built a “Contact Jacobian” which we then used in
our solver loop. For each of our joint and motor constraints, we’ll follow the
same pattern. We’ll define a setup step, and a solve step.

Once we are done, we’ll build out a suite of APIs to help construct the
constraint data from higher-level concepts.

We have 6 constraints to work through, so let’s get started.

## The Position Constraint

Unity calls this “Linear”, however, I think that’s a poor choice since that term
refers to velocity, and this rule specifies a relationship of positions. Thus,
we’ll call ours the Position Constraint.

### Position Constraint Solve

We’ll start by manually porting over the Solve method inside
LinearLimitJacobian.cs from Unity Physics. Right off the bat, this method
computes the inertial pose world transforms for the end of the timestep. This is
essentially integration, but without the damping part. We should add a method
for this, since we’ll probably need it quite often.

```csharp
/// <summary>
/// Computes a new world transform of an object using its velocity and the specified time step
/// </summary>
/// <param name="inertialPoseWorldTransform">The original world transform of the center of mass and inertia tensor diagonal before the time step</param>
/// <param name="velocity">The linear and angular velocity to apply to the world transform</param>
/// <param name="deltaTime">The time step over which the velocity should be applied</param>
/// <returns>A new inertialPoseWorldTransform propagated by the Velocity across the time span of deltaTime</returns>
public static RigidTransform IntegrateWithoutDamping(RigidTransform inertialPoseWorldTransform, in Velocity velocity, float deltaTime)
{
    inertialPoseWorldTransform.pos += velocity.linear * deltaTime;
    var halfDeltaAngle = velocity.angular * 0.5f * deltaTime;
    var dq = new quaternion(new float4(halfDeltaAngle, 1f));
    inertialPoseWorldTransform.rot = math.normalize(math.mul(inertialPoseWorldTransform.rot, dq));
    return inertialPoseWorldTransform;
}
```

This also means our solve method will need mutable velocities as well as the
inertial pose world transforms and delta time.

Next, Unity Physics desires a `BodyFromContraintsA.Translation`. This is
actually the joint position relative to the inertial pose. We’ll make this an
argument too. This gets used in a method called `CalculateAngulars()`, which is
exclusive to this method. Unity has it as a member method of a struct. We’ll
make it a local method.

So far, we have this:

```csharp
public static void SolveJacobian(ref Velocity velocityA, in RigidTransform inertialPoseWorldTransformA, float3 jointPositionInInertialPoseASpace,
                                    ref Velocity velocityB, in RigidTransform inertialPoseWorldTransformB, float3 jointPositionInInertialPoseBSpace,
                                    float deltaTime)
{
    var futureTransformA = IntegrateWithoutDamping(inertialPoseWorldTransformA, in velocityA, deltaTime);
    var futureTransformB = IntegrateWithoutDamping(inertialPoseWorldTransformB, in velocityB, deltaTime);

    // Calculate the angulars
    CalculateAngulars(jointPositionInInertialPoseASpace, futureTransformA.rot, out float3 angA0, out float3 angA1, out float3 angA2);
    CalculateAngulars(jointPositionInInertialPoseBSpace, futureTransformB.rot, out float3 angB0, out float3 angB1, out float3 angB2);

    static void CalculateAngulars(float3 pivotInMotion, quaternion worldFromMotionRotation, out float3 ang0, out float3 ang1, out float3 ang2)
    {
        // Jacobian directions are i, j, k
        // Angulars are pivotInMotion x (motionFromWorld * direction)
        float3x3 motionFromWorldRotation = math.transpose(new float3x3(worldFromMotionRotation));
        ang0                             = math.cross(pivotInMotion, motionFromWorldRotation.c0);
        ang1                             = math.cross(pivotInMotion, motionFromWorldRotation.c1);
        ang2                             = math.cross(pivotInMotion, motionFromWorldRotation.c2);
    }
}
```

Next, we need to calculate an “effective mass matrix”. I’ll be honest, I haven’t
quite figured out what that means yet. Though some of the methods used for this
we’ve already ported for contacts. Since they will be shared, it would be wise
to move them to a new file. UnitySimJacobianUtilities.cs is the home.

We also need the inverse mass and inertia for these, so that’s more arguments to
bring in.

Moving along, now we need to calculate an error value. This represents how far
off we are from satisfying the constraint. Unity Physics defines this as a
member method of the struct. It would be tempting to inline it, but we’ll
actually need this method for building too. So we’ll make it a private static
method. This method does access some special fields exclusive to the constraint
pair. We’ll put all of these into a `PositionConstraintJacobianParameters`.

We also need a couple more static general-purpose constraint solving methods in
our new file. One of these is a method to compute the correction based on the
error, incorporating some extra constraint-specific arguments named `tau` and
`damping`. We’ll also need the `inverseDeltaTime` as an argument, just like with
contacts.

And now, we have a correction impulse. This impulse is in world-space as applied
to A, with B being the negated value. Unity Physics has some special logic to
convert it into “constraint space” for events to match Havok behavior. I have no
idea what “constraint space” means. And there’s no example of its usage in the
samples. But a world-space impulse seems pretty useful if we want joint-breaking
logic for when the impulse becomes too strong. We’ll make that the return value.

We also need to apply the impulse to our velocity. This becomes another local
static method.

Here’s everything so far:

```csharp
public struct PositionConstraintJacobianParameters
{
    // Pivot distance limits
    public float minDistance;
    public float maxDistance;

    // If the constraint limits 1 DOF, this is the constrained axis.
    // If the constraint limits 2 DOF, this is the free axis.
    // If the constraint limits 3 DOF, this is unused and set to float3.zero
    public float3 axisInB;

    // Position error at the beginning of the step
    public float initialError;

    // Fraction of the position error to correct per step
    public float tau;

    // Fraction of the velocity error to correct per step
    public float damping;

    // True if the jacobian limits one degree of freedom
    public bool is1D;
}

public static float3 SolveJacobian(ref Velocity velocityA, in RigidTransform inertialPoseWorldTransformA, float3 jointPositionInInertialPoseASpace, in Mass massA,
                                    ref Velocity velocityB, in RigidTransform inertialPoseWorldTransformB, float3 jointPositionInInertialPoseBSpace, in Mass massB,
                                    in PositionConstraintJacobianParameters parameters, float deltaTime, float inverseDeltaTime)
{
    var futureTransformA = IntegrateWithoutDamping(inertialPoseWorldTransformA, in velocityA, deltaTime);
    var futureTransformB = IntegrateWithoutDamping(inertialPoseWorldTransformB, in velocityB, deltaTime);

    // Calculate the angulars
    CalculateAngulars(jointPositionInInertialPoseASpace, futureTransformA.rot, out float3 angA0, out float3 angA1, out float3 angA2);
    CalculateAngulars(jointPositionInInertialPoseBSpace, futureTransformB.rot, out float3 angB0, out float3 angB1, out float3 angB2);

    // Calculate effective mass
    float3 effectiveMassDiag, effectiveMassOffDiag;
    {
        // Calculate the inverse effective mass matrix
        float3 invEffectiveMassDiag = new float3(
            CalculateInvEffectiveMassDiag(angA0, massA.inverseInertia, massA.inverseMass, angB0, massB.inverseInertia, massB.inverseMass),
            CalculateInvEffectiveMassDiag(angA1, massA.inverseInertia, massA.inverseMass, angB1, massB.inverseInertia, massB.inverseMass),
            CalculateInvEffectiveMassDiag(angA2, massA.inverseInertia, massA.inverseMass, angB2, massB.inverseInertia, massB.inverseMass));

        float3 invEffectiveMassOffDiag = new float3(
            CalculateInvEffectiveMassOffDiag(angA0, angA1, massA.inverseInertia, angB0, angB1, massB.inverseInertia),
            CalculateInvEffectiveMassOffDiag(angA0, angA2, massA.inverseInertia, angB0, angB2, massB.inverseInertia),
            CalculateInvEffectiveMassOffDiag(angA1, angA2, massA.inverseInertia, angB1, angB2, massB.inverseInertia));

        // Invert to get the effective mass matrix
        InvertSymmetricMatrix(invEffectiveMassDiag, invEffectiveMassOffDiag, out effectiveMassDiag, out effectiveMassOffDiag);
    }

    // Predict error at the end of the step and calculate the impulse to correct it
    float3 impulse;
    {
        // Find the difference between the future distance and the limit range, then apply tau and damping
        float futureDistanceError = CalculatePositionConstraintError(in parameters,
                                                                        futureTransformA,
                                                                        jointPositionInInertialPoseASpace,
                                                                        futureTransformB,
                                                                        jointPositionInInertialPoseBSpace,
                                                                        out float3 futureDirection);
        float solveDistanceError = CalculateCorrection(futureDistanceError, parameters.initialError, parameters.tau, parameters.damping);

        // Calculate the impulse to correct the error
        float3   solveError    = solveDistanceError * futureDirection;
        float3x3 effectiveMass = BuildSymmetricMatrix(effectiveMassDiag, effectiveMassOffDiag);
        impulse                = math.mul(effectiveMass, solveError) * inverseDeltaTime;
    }
    // Apply the impulse
    ApplyImpulse(ref velocityA, in massA, impulse,  angA0, angA1, angA2);
    ApplyImpulse(ref velocityB, in massB, -impulse, angB0, angB1, angB2);
    return impulse;

    static void CalculateAngulars(float3 pivotInMotion, quaternion worldFromMotionRotation, out float3 ang0, out float3 ang1, out float3 ang2)
    {
        // Jacobian directions are i, j, k
        // Angulars are pivotInMotion x (motionFromWorld * direction)
        float3x3 motionFromWorldRotation = math.transpose(new float3x3(worldFromMotionRotation));
        ang0                             = math.cross(pivotInMotion, motionFromWorldRotation.c0);
        ang1                             = math.cross(pivotInMotion, motionFromWorldRotation.c1);
        ang2                             = math.cross(pivotInMotion, motionFromWorldRotation.c2);
    }

    static void ApplyImpulse(ref Velocity velocity, in Mass mass, in float3 impulse, in float3 ang0, in float3 ang1, in float3 ang2)
    {
        velocity.linear       += impulse * mass.inverseMass;
        float3 angularImpulse  = impulse.x * ang0 + impulse.y * ang1 + impulse.z * ang2;
        velocity.angular      += angularImpulse * mass.inverseInertia;
    }
}

static float CalculatePositionConstraintError(in PositionConstraintJacobianParameters parameters,
                                                in RigidTransform inertialPoseWorldTransformA, float3 jointPositionInInertialPoseASpace,
                                                in RigidTransform inertialPoseWorldTransformB, float3 jointPositionInInertialPoseBSpace,
                                                out float3 direction)
{
    // Find the direction from pivot A to B and the distance between them
    float3 pivotA = math.transform(inertialPoseWorldTransformA, jointPositionInInertialPoseASpace);
    float3 pivotB = math.transform(inertialPoseWorldTransformB, jointPositionInInertialPoseBSpace);
    float3 axis   = math.mul(inertialPoseWorldTransformB.rot, parameters.axisInB);
    direction     = pivotB - pivotA;
    float dot     = math.dot(direction, axis);

    // Project for lower-dimension joints
    float distance;
    if (parameters.is1D)
    {
        // In 1D, distance is signed and measured along the axis
        distance  = -dot;
        direction = -axis;
    }
    else
    {
        // In 2D / 3D, distance is nonnegative.  In 2D it is measured perpendicular to the axis.
        direction               -= axis * dot;
        float futureDistanceSq   = math.lengthsq(direction);
        float invFutureDistance  = math.select(math.rsqrt(futureDistanceSq), 0.0f, futureDistanceSq == 0.0f);
        distance                 = futureDistanceSq * invFutureDistance;
        direction               *= invFutureDistance;
    }

    // Find the difference between the future distance and the limit range
    return CalculateError(distance, parameters.minDistance, parameters.maxDistance);
}
```

### Position Constraint Build

Our goal now is to generate `PositionConstraintJacobianParameters`. However, 3.5
of 7 parameters are actually inputs coming from Unity’s Constraint type or the
solver context.

Wait, 3.5?

One of the values is our `jointPositionInInertialPoseBSpace`. This value gets
passed in, and then is modified to store the projection of the joint along
within the constrained axes, removing the dimensionality from the unconstrained
axes. There’s also a big TODO in Unity Physics detailing an issue with this
projection. Basically, this constraint allows for constraining two objects
within a distance of each other. When that constraint is not zero, and both
bodies are dynamic, the projection isn’t spatially accurate and “may not look
correct”. However, the common cases for this constraint are all correct. We’ll
have to keep this limitation in mind, and either wait for Unity to solve it or
solve it ourselves in the future.

Because of this projection, we need the orientation of the joint for B, but not
A. We’ll also move these joint positions into our parameters struct.

Two of our inputs define the min and max distance the joint points are allowed
to be apart along the constrained axes. If two axes are constrained, then we get
a cylindrical volume of free movement.

The other two inputs are tau and damping. These get computed from other values
including the timestep and the solver iteration count. If these latter two
values are constant, then tau and damping could be cached. For this reason, we
keep them as inputs, and we’ll discuss how to calculate them later.

Lastly, we need a `bool3` representing which axes are constrained.

Omitting the giant TODO comment, we end up with this:

```csharp
public static void BuildJacobian(out PositionConstraintJacobianParameters parameters,
                                    in RigidTransform inertialPoseWorldTransformA, float3 jointPositionInInertialPoseASpace,
                                    in RigidTransform inertialPoseWorldTransformB, in RigidTransform jointTransformInInertialPoseBSpace,
                                    float minDistance, float maxDistance, float tau, float damping, bool3 constrainedAxes)
{
    parameters = default;

    parameters.minDistance                       = minDistance;
    parameters.maxDistance                       = maxDistance;
    parameters.tau                               = tau;
    parameters.damping                           = damping;
    parameters.jointPositionInInertialPoseASpace = jointPositionInInertialPoseASpace;
    parameters.jointPositionInInertialPoseBSpace = jointTransformInInertialPoseBSpace.pos;

    if (!math.all(constrainedAxes))
    {
        parameters.is1D = constrainedAxes.x ^ constrainedAxes.y ^ constrainedAxes.z;

        // Project pivot A onto the line or plane in B that it is attached to
        RigidTransform bFromA     = math.mul(math.inverse(inertialPoseWorldTransformB), inertialPoseWorldTransformA);
        float3         pivotAinB  = math.transform(bFromA, jointPositionInInertialPoseASpace);
        float3         diff       = pivotAinB - jointTransformInInertialPoseBSpace.pos;
        var            jointBAxes = new float3x3(jointTransformInInertialPoseBSpace.rot);
        for (int i = 0; i < 3; i++)
        {
            float3 column      = jointBAxes[i];
            parameters.axisInB = math.select(column, parameters.axisInB, parameters.is1D ^ constrainedAxes[i]);

            float3 dot                                    = math.select(math.dot(column, diff), 0.0f, constrainedAxes[i]);
            parameters.jointPositionInInertialPoseBSpace += column * dot;
        }
    }

    // Calculate the current error
    parameters.initialError = CalculatePositionConstraintError(in parameters, in inertialPoseWorldTransformA, parameters.jointPositionInInertialPoseASpace,
                                                                in inertialPoseWorldTransformB, parameters.jointPositionInInertialPoseBSpace, out _);
}
```

Phew. One constraint down. 5 more to go!

## The Rotation Constraint

Todo: This is where I am going to cut things off for now. But don’t worry, I’m
not giving up on this. Just need to take things in stride.
