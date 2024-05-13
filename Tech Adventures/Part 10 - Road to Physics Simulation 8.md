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
this functionality, and we’ll begin with the essentials for rigid bodies:
joints.

## What Even Are Joints?

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

Lastly, we need a `bool3` representing which axes are constrained. These are
relative to the shared axes of the joint.

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
    parameters.initialError = CalculatePositionConstraintError(in parameters, in inertialPoseWorldTransformA, in inertialPoseWorldTransformB, out _);
}
```

Phew. One constraint down. 5 more to go!

## The Rotation 1D Constraint

Did I say 5? Sorry. I meant 7. Rotation constraints (Angular Limit in Unity
Physics) has three variants, one for each dimensionality. Unlike the position
constraint which could mostly generalize across all three dimensions, rotations
require wildly different logic at each dimension.

### Rotation 1D Constraint Solve

We start off by integrating our bodies. That means we need the velocities as
well as the timestep. However, there’s a twist this time. Instead of integrating
in world-space, Unity Physics chooses to only integrate the rotations relative
to each other. This apparently is done for performance benefits. We can’t say no
to that!

To do this, we need a relative rotation, `inertialPoseAInInertialPoseBSpace`.
That will go into a `Rotation1DConstraintJacobianParameters` struct. And we need
a new utility method in our general Jacobian Utilities file to perform the
integration. Here’s my version of that method, tweaked to help Burst out a
little:

```csharp
static quaternion IntegrateOrientationBFromA(quaternion bFromA, float3 angularVelocityA, float3 angularVelocityB, float timestep)
{
    var halfDeltaTime = timestep * 0.5f;
    var dqA           = new quaternion(new float4(angularVelocityA * halfDeltaTime, 1f));
    var invDqB        = new quaternion(new float4(angularVelocityB * -halfDeltaTime, 1f));
    return math.normalize(math.mul(math.mul(invDqB, bFromA), dqA));
}
```

Next, we need a parameter in our parameters struct named
`axisInInertialPoseASpace` to calculate the effective mass.

Now, we need to compute our error and its correction just like with position
constraints. The error correction logic has a note that for 1D, the algorithm
only works well for angles constrained to 90 degrees or less, because otherwise
there’s multiple solutions. With multiple solutions, a stateless engine could
pick different solutions every frame, resulting in a glitchy mess. There’s a
method inside this error correction called `CalculateTwistAngle()` which we will
need to add to our utilities.

Lastly, we have to compute the correction and apply the impulse. This time, the
impulse takes the form of a scalar value which then gets applied to each body as
a rotation-only impulse.

Here’s what we end up with, with the giant pitfall comment removed again:

```csharp
public struct Rotation1DConstraintJacobianParameters
{
    public quaternion inertialPoseAInInertialPoseBSpace;
    public quaternion jointRotationInInertialPoseASpace;
    public quaternion jointRotationInInertialPoseBSpace;

    // Limited axis in motion A space
    public float3 axisInInertialPoseASpace;

    public float minAngle;
    public float maxAngle;

    public float initialError;
    public float tau;
    public float damping;

    public int axisIndex;
}

// Returns the scalar impulse applied only to the angular velocity for the constrained axis.
public static float SolveJacobian(ref Velocity velocityA, in Mass massA, ref Velocity velocityB, in Mass massB,
                                    in Rotation1DConstraintJacobianParameters parameters, float deltaTime, float inverseDeltaTime)
{
    // Predict the relative orientation at the end of the step
    quaternion futureMotionBFromA = IntegrateOrientationBFromA(parameters.inertialPoseAInInertialPoseBSpace, velocityA.angular, velocityB.angular, deltaTime);

    // Calculate the effective mass
    float3 axisInMotionB = math.mul(futureMotionBFromA, -parameters.axisInInertialPoseASpace);
    float  effectiveMass;
    {
        float invEffectiveMass = math.csum(parameters.axisInInertialPoseASpace * parameters.axisInInertialPoseASpace * massA.inverseInertia +
                                            axisInMotionB * axisInMotionB * massB.inverseInertia);
        effectiveMass = math.select(1.0f / invEffectiveMass, 0.0f, invEffectiveMass == 0.0f);
    }

    // Calculate the error, adjust by tau and damping, and apply an impulse to correct it
    float futureError  = CalculateRotation1DConstraintError(in parameters, futureMotionBFromA);
    float solveError   = CalculateCorrection(futureError, parameters.initialError, parameters.tau, parameters.damping);
    float impulse      = math.mul(effectiveMass, -solveError) * inverseDeltaTime;
    velocityA.angular += impulse * parameters.axisInInertialPoseASpace * massA.inverseInertia;
    velocityB.angular += impulse * axisInMotionB * massB.inverseInertia;

    return impulse;
}

static float CalculateRotation1DConstraintError(in Rotation1DConstraintJacobianParameters parameters, quaternion motionBFromA)
{
    // Calculate the relative joint frame rotation
    quaternion jointBFromA = math.mul(math.InverseRotateFast(parameters.jointRotationInInertialPoseBSpace, motionBFromA), parameters.jointRotationInInertialPoseASpace);

    // Find the twist angle of the rotation.
    float angle = CalculateTwistAngle(jointBFromA, parameters.axisIndex);

    // Angle is in [-2pi, 2pi].
    // For comparison against the limits, find k so that angle + 2k * pi is as close to [min, max] as possible.
    float centerAngle = (parameters.minAngle + parameters.maxAngle) / 2.0f;
    bool  above       = angle > (centerAngle + math.PI);
    bool  below       = angle < (centerAngle - math.PI);
    angle             = math.select(angle, angle - 2.0f * math.PI, above);
    angle             = math.select(angle, angle + 2.0f * math.PI, below);

    // Calculate the relative angle about the twist axis
    return CalculateError(angle, parameters.minAngle, parameters.maxAngle);
}
```

### Rotation 1D Constraint Build

There is absolutely nothing new here. There’s a few values that are trivially
calculated, and the rest comes from arguments.

```csharp
public static void BuildJacobian(out Rotation1DConstraintJacobianParameters parameters,
                                    quaternion inertialPoseWorldRotationA, quaternion jointRotationInInertialPoseASpace,
                                    quaternion inertialPoseWorldRotationB, quaternion jointRotationInInertialPoseBSpace,
                                    float minAngle, float maxAngle, float tau, float damping, int axisIndex)
{
    parameters = new Rotation1DConstraintJacobianParameters
    {
        inertialPoseAInInertialPoseBSpace = math.normalize(math.InverseRotateFast(inertialPoseWorldRotationB, inertialPoseWorldRotationA)),
        jointRotationInInertialPoseASpace = jointRotationInInertialPoseASpace,
        jointRotationInInertialPoseBSpace = jointRotationInInertialPoseBSpace,
        axisInInertialPoseASpace          = new float3x3(jointRotationInInertialPoseASpace)[axisIndex],
        minAngle                          = minAngle,
        maxAngle                          = maxAngle,
        tau                               = tau,
        damping                           = damping,
        axisIndex                         = axisIndex
    };
    parameters.initialError = CalculateRotation1DConstraintError(in parameters, parameters.inertialPoseAInInertialPoseBSpace);
}
```

## The Rotation 2D Constraint

Double the dimensions, double the problems, right?

Well, hopefully not. 2D is kinda a mix of 1D, some more complicated math we
won’t investigate, and a very small amount of new things.

### Rotation 2D Constraint Solve

Just like in 1D, our 2D variant integrates rotations in relative space. We can
use the same arguments as 1D, except with a different `parameters` struct.

This time, we’ll need an axis from each inertial pose space, but we won’t need
the joint rotations in that space anymore, which is nice.

The next curveball is the presence of `RSqrtSafe()`. This is a new utility
define like so:

```csharp
static float RSqrtSafe(float v) => math.select(math.rsqrt(v), 0.0f, math.abs(v) < 1e-10f);
```

Why `1e-10f`? I have no idea. That’s probably why the method is internal in
Unity Physics.

The rest of the method can be ported over without any incident, all the way up
to where we compute the impulse. This impulse is a 2D impulse, with a value for
each of the constrained axes and is applied exclusively to the angular velocity.
We’ll need to provide a method for the user to map these values to ordinate
indices.

It all looks like this:

```csharp
public struct Rotation2DConstraintJacobianParameters
{
    public quaternion inertialPoseAInInertialPoseBSpace;

    public float3 axisAInInertialPoseASpace;
    public float3 axisBInInertialPoseBSpace;

    public float minAngle;
    public float maxAngle;

    public float initialError;
    public float tau;
    public float damping;
}

public static int2 ConvertRotation2DJacobianFreeRotationIndexToImpulseIndices(int freeIndex) => (freeIndex + new int2(1, 2)) % 3;

// Returns the scalar impulse applied only to the angular velocity for the constrained axis.
public static float2 SolveJacobian(ref Velocity velocityA, in Mass massA, ref Velocity velocityB, in Mass massB,
                                    in Rotation2DConstraintJacobianParameters parameters, float deltaTime, float inverseDeltaTime)
{
    // Predict the relative orientation at the end of the step
    quaternion futureBFromA = IntegrateOrientationBFromA(parameters.inertialPoseAInInertialPoseBSpace, velocityA.angular, velocityB.angular, deltaTime);

    // Calculate the jacobian axis and angle
    float3 axisAinB     = math.mul(futureBFromA, parameters.axisAInInertialPoseASpace);
    float3 jacB0        = math.cross(axisAinB, parameters.axisBInInertialPoseBSpace);
    float3 jacA0        = math.mul(math.inverse(futureBFromA), -jacB0);
    float  jacLengthSq  = math.lengthsq(jacB0);
    float  invJacLength = RSqrtSafe(jacLengthSq);
    float  futureAngle;
    {
        float sinAngle = jacLengthSq * invJacLength;
        float cosAngle = math.dot(axisAinB, parameters.axisBInInertialPoseBSpace);
        futureAngle    = math.atan2(sinAngle, cosAngle);
    }

    // Choose a second jacobian axis perpendicular to A
    float3 jacB1 = math.cross(jacB0, axisAinB);
    float3 jacA1 = math.mul(math.inverse(futureBFromA), -jacB1);

    // Calculate effective mass
    float2 effectiveMass;  // First column of the 2x2 matrix, we don't need the second column because the second component of error is zero
    {
        // Calculate the inverse effective mass matrix, then invert it
        float invEffMassDiag0   = math.csum(jacA0 * jacA0 * massA.inverseInertia + jacB0 * jacB0 * massB.inverseInertia);
        float invEffMassDiag1   = math.csum(jacA1 * jacA1 * massA.inverseInertia + jacB1 * jacB1 * massB.inverseInertia);
        float invEffMassOffDiag = math.csum(jacA0 * jacA1 * massA.inverseInertia + jacB0 * jacB1 * massB.inverseInertia);
        float det               = invEffMassDiag0 * invEffMassDiag1 - invEffMassOffDiag * invEffMassOffDiag;
        float invDet            = math.select(jacLengthSq / det, 0.0f, det == 0.0f);  // scale by jacLengthSq because the jacs were not normalized
        effectiveMass           = invDet * new float2(invEffMassDiag1, -invEffMassOffDiag);
    }

    // Normalize the jacobians
    jacA0 *= invJacLength;
    jacB0 *= invJacLength;
    jacA1 *= invJacLength;
    jacB1 *= invJacLength;

    // Calculate the error, adjust by tau and damping, and apply an impulse to correct it
    float  futureError  = CalculateError(futureAngle, parameters.minAngle, parameters.maxAngle);
    float  solveError   = CalculateCorrection(futureError, parameters.initialError, parameters.tau, parameters.damping);
    float2 impulse      = -effectiveMass * solveError * inverseDeltaTime;
    velocityA.angular  += massA.inverseInertia * (impulse.x * jacA0 + impulse.y * jacA1);
    velocityB.angular  += massB.inverseInertia * (impulse.x * jacB0 + impulse.y * jacB1);

    return impulse;
}
```

### Rotation 2D Constraint Build

If there’s any surprise this time, it is that Unity Physics doesn’t have a
dedicated error calculation method for this constraint type, and instead just
calculates it inline.

```csharp
public static void BuildJacobian(out Rotation2DConstraintJacobianParameters parameters,
                                    quaternion inertialPoseWorldRotationA, quaternion jointRotationInInertialPoseASpace,
                                    quaternion inertialPoseWorldRotationB, quaternion jointRotationInInertialPoseBSpace,
                                    float minAngle, float maxAngle, float tau, float damping, int freeAxisIndex)
{
    parameters = new Rotation2DConstraintJacobianParameters
    {
        inertialPoseAInInertialPoseBSpace = math.normalize(math.InverseRotateFast(inertialPoseWorldRotationB, inertialPoseWorldRotationA)),
        axisAInInertialPoseASpace         = new float3x3(jointRotationInInertialPoseASpace)[freeAxisIndex],
        axisBInInertialPoseBSpace         = new float3x3(jointRotationInInertialPoseBSpace)[freeAxisIndex],
        minAngle                          = minAngle,
        maxAngle                          = maxAngle,
        tau                               = tau,
        damping                           = damping,
    };
    // Calculate the initial error
    {
        float3 axisAinB         = math.mul(parameters.inertialPoseAInInertialPoseBSpace, parameters.axisAInInertialPoseASpace);
        float  sinAngle         = math.length(math.cross(axisAinB, parameters.axisBInInertialPoseBSpace));
        float  cosAngle         = math.dot(axisAinB, parameters.axisBInInertialPoseBSpace);
        float  angle            = math.atan2(sinAngle, cosAngle);
        parameters.initialError = CalculateError(angle, parameters.minAngle, parameters.maxAngle);
    }
}
```

## The Rotation 3D Constraint

The 3D constraint operates a lot like the 2D constraint, except there’s a lot
more inline matrix math involved. And as you might expect, it generates a 3D
impulse, of which there is no ambiguity as to what each component means.

In fact, the only real tricky part about this version is one of the struct
parameters Unity named `RefBFromA`, alongside the normal `BFromA` which we
translated to `inertialPoseAInInertialPoseBSpace`. It seems to be some kind of
bind orientation between the two bodies at the joint, to map the inertial pose
from one body to the compensated inertial pose of the other.

Regardless of what it is called, Unity Physics only ever uses the inverse of
this value, so as a tiny optimization, we’ll store the inverse of this rotation.

### Breather Time

We are done with the non-motor constraints. We’ve been able to successfully
reason about all the inputs and outputs to these constraints. And we’ve
developed an understanding of their shortcomings. However, the math describing
the generation of the impulses with things like “effective masses” and the
matrices still elude us. It is possible this math is the real reason why Unity
Physics calls all these things “Jacobians”. Perhaps with motors, we’ll gain some
clues for how to write custom constraints?

Otherwise, we at least these building blocks on which to build our joints. But
before we do that, let’s try to make sense of motors, just to make sure we don’t
miss anything in our foundation.

## The Position 1D Motor Or What Are We Doing?

Wait, 1D?

Oh no, does that mean that we have three of these as well?

Nope.

Unity Physics only supports 1D position motors. We can all take a sigh of
relief.

But why does Unity Physics only support 1D motors?

Architecture.

Oh no.

The way the position motor works in Unity Physics is that BodyA in the
constraint is the motorized body, and the BodyB is the target. BodyB has an axis
extruding from the joint point, and the BodyA’s joint point is projected onto
this axis. The distance between the projected point and BodyB’s joint point is
the compared against a target distance, to create an impulse along the axis. The
impulse is then clamped based on a max impulse which is accumulated between
solver iterations.

In this case, the motor drives towards a plane. But what if we want to drive
towards a line, or a point, or a volume?

The majority of the logic is nearly identical to the position constraint. So
much so, that even without fully understanding the effective mass matrix stuff,
we could still probably implement these, but it brings to question which of
these are the common cases, and which of these are the rare ones? What is worth
generalizing? And what is worth being specific and concise with our API?

With all these unanswered questions, perhaps we should ignore motors for now,
and explore them in a future adventure.

Anyways, let’s move on to computing our `tau` and `damping` for our current
constraints.

## Springs and Dampers

Have you ever heard of Hooke’s Law? It has a formula like this:

F = -kx

Here, F is a force of a spring-like structure, x is how much the spring-like
structure is expanded or contracted beyond its resting state, and k is the
spring constant, where a higher value results in a stiffer spring. The negative
sign is because the force opposes the displacement

A spring damper represents internal frictional or friction-like forces that
resist the spring. Thus, a spring damper equation is as follows:

F = -kx - cv

In this equation, v is the velocity and c is the damping constant. Thus, a
damped spring can be characterized by its spring constant and its damping
constant.

If we wanted a spring that had the same motion behavior independent of masses,
we can instead represent it as a spring frequency and a damping ratio. Here’s
some code to convert between representations:

```csharp
public static float kStiffSpringFrequency = 74341.31f;
public static float kStiffDampingRatio    = 2530.126f;

public static float SpringFrequencyFrom(float springConstant, float inverseMass)
{
    return springConstant * inverseMass * rcpTwoPI;
}

public static float SpringConstantFrom(float springFrequency, float mass)
{
    return springFrequency * mass * 2f * math.PI;
}

public static float DampingRatioFrom(float springConstant, float dampingConstant, float mass)
{
    var product = springConstant * mass;
    if (product < math.EPSILON)
    {
        if (springConstant < math.EPSILON || mass < math.EPSILON)
            return kStiffDampingRatio;

        var critical = 2f * math.sqrt(springConstant) * math.sqrt(mass);
        return dampingConstant / critical;
    }
    return dampingConstant / (2f * math.sqrt(product));  // damping coefficient / critical damping coefficient
}

public static float DampingConstantFrom(float springConstant, float dampingRatio, float mass)
{
    var product = springConstant * mass;
    if (product < math.EPSILON)
    {
        if (springConstant < math.EPSILON || mass < math.EPSILON)
            return dampingRatio * float.Epsilon;
        var critical = 2f * math.sqrt(springConstant) * math.sqrt(mass);
        return dampingRatio * critical;
    }
    return dampingRatio * 2f * math.sqrt(product);
}

const float rcpTwoPI = 0.5f / math.PI;
```

Note that for damping ratio, there’s a failure case for small values. My version
handles the approximation of small values a little more elegantly than Unity
Physics.

You’ll also see some “stiff” constants, which describe a really, really,
*really* stiff spring. In fact, this is the spring that Unity uses for “rigid”
joints.

So what do all of these springs and dampers have to do with constraints? Well,
there’s a giant comment block in Unity Physics which starts with the following:

>   In the following we derive the formulas for converting spring frequency and
>   damping ratio to the solver constraint regularization parameters tau and
>   damping, representing a normalized stiffness factor and damping factor,
>   respectively.

Turns out that alongside a time step and the number of solver iterations, you
can convert these spring parameters into constraint parameters and vice-versa,
thanks to a very good derivation from the Unity Physics team.

I won’t go into details on how these conversions work. We can pretty much copy
them as-is. There are probably some micro-optimization opportunities in the
future, but that would be for Optimization Adventures.

## Composing Joints from Constraints

Now that we have a mechanism to build and solve constraint Jacobians, and we
have the ability to specify the constraint springs and dampers, the final step
is to compose actual joints from these parts.

Unity Physics provides many high-level utility APIs for representing joints
found in PhysX and other engines. However, these APIs assume a runtime data
storage representation. However, if you’ve decided to delve into the control and
flexibility Psyshock has to offer, you may find that constraints are not that
difficult to reason about.

For example, to create a hinge joint for a door, you would first want to lock
the position of the door on all axes, as the door can only rotate. That’s done
by setting the min and max values to zero and using the really stiff spring
constants. You’d want to do the same for rotation on two axes using a 2D
rotation constraint. Then the third constraint would be a 1D rotation constraint
with a nonzero max value, and optionally a less stiff spring if you want to
allow a little overshoot.

Remember, for the rotation constraints, the jointRotationInInertialPoseA/BSpace
should match when transform to world space when the pair of bodies are in their
initial relative orientations. You can rotate both of these in world-space to
change the axes of rotation. This allows you to have multiple rotation
constraints around different axes which can be used for asymmetric joints like
shoulders.

As another example, perhaps instead of a rotating door we have a sliding door.
This time, we want to lock the position on two axes, and lock rotation on all
three. Then, we want a distance constraint with a min and max along our
free-moving axis.

## Incorporating Constraints

Unfortunately, I don’t have a good project for demonstrating joints yet. And I
would prefer to keep Free Parking’s demo simple. But I can provide a few hints
for how you might incorporate all of this.

First, you need to add both bodies of a constraint to a `PairStream`. This time,
you can’t use a `ParallelWriter`, and instead have to provide the bucket indices
yourself. To get the bucket indices, you can calculate them from an `Aabb` using
`CollisionLayerBucketIndexCalculator`. If one of the bodies is static, make sure
`isRW` is `false` for that body, and then you can pass in the same bucket index
as the dynamic body. It doesn’t matter what bucket index the static body
actually belongs to. For world constraints (such as global orientation locks),
you can use `Entity.Null` for the non-body in the pair.

Next, you need a way to encode the constraint Jacobians into the pair. You can
use `userUShort` and `userByte` to encode the base type information that you
write. You may wish to pack multiple constraints into a single type struct for
common joint types. This way, you can avoid having to load entity data multiple
times when processing multiple constraints for the same pair.

Lastly, since you can’t use `ParallelWriter`, you must add these constraints in
single-threaded jobs. To regain some level of parallelism, you can create a
separate `PairStream` using the same allocator for each ECS query with
constraints. Then later in an `IJob`, you can concatenate the `PairStreams` via
`PairStream.ConcatenateFrom()`. This will allow you to combine all your
`PairStreams` from various FindPairs operations and all your constraint
`PairStreams` into a single `PairStream` which you can use in a single
`Physics.ForEachPair()` in your solver loop.

## What’s Next

We now have constraints which can model joints. This makes a lot of cool new
physics things possible. But before we dive further, we’ll likely need some more
real-world use cases to help us explore all that the world of physics engines
has to offer.

I don’t know what the next adventure will entail, but I do know there will be
another one. In the meantime, be sure to try out all these rigid body APIs
yourself and see if you can build the perfect physics engine for your game!

Thanks for reading!
