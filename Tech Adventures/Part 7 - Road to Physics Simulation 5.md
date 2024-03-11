# Tech Adventures: Road to Physics Simulation Part 5

You know that point in an adventure game where you’ve gone through the grind,
and you’ve already beaten what is considered the most difficult boss in the game
relative to your progression, and now there’s a bunch of small quests and
storylines before you take on the final challenge that culminates with the final
boss? That’s where we are at right now.

In this part of our adventure, we’ll explore the pieces that relate to motion
and integration. We’ll need to bake out mass data to give our rigid bodies
properly scaled reactions to forces. We’ll need to compute motion-expanded AABBs
so that FindPairs can anticipate collisions. And finally, we will write the
integrator logic that allows us to compute the final positions and velocities of
our rigid bodies.

Let’s get started!

## Masses, Center of Mass, and Angular Expansions

Unity Physics combines mass data with a concept called “angular expansion”.
Angular expansion represents how much an AABB could possibly expand by when the
object is rotated. This is more continuously conservative estimate compared to
simply transforming an AABB as is currently done with `Physics.AabbFrom()`.
While we’ll keep this data separated from mass data, we’ll figure out how to
compute them all together since they are collider-specific.

What is also collider-specific is rotational inertia. Unlike normal mass which
doesn’t really depend on the shape to figure out how much acceleration happens
with a force, rotational inertia is dependent on the shape. Unity Physics
represents rotational inertia as a 3-component vector representing the
resistance around each local axis. We’ll do the same.

Speaking of, Unity Physics has a tendency to call rotational inertia just
“inertia”, which is a bit of a stretch of the definition. However, PhysX,
Bullet, and Jolt also use this simplified name, so I guess this is a common
convention.

We’ll also need to calculate the center of mass for each collider type, so we’ll
figure that out as well.

### Sphere Colliders

The center of mass of a sphere is easy. It is simply the center of the sphere.

```csharp
public static float3 LocalCenterOfMassFrom(in Collider collider)
{
    switch (collider.type)
    {
        case ColliderType.Sphere:
            return collider.m_sphere.center;
```

Additionally, the sphere never rotates about its center of mass. So its angular
expansion is simply 0.

The inertia value is identical for all three axes and each component can be
computed from the formula:

```csharp
return (2f / 5f) * collider.m_sphere.radius * collider.m_sphere.radius;
```

That might seem like a weird formula, but you can find its derivation
[here](http://hyperphysics.phy-astr.gsu.edu/hbase/isph.html).

Of course, this isn’t the full rotational inertia, but rather the normalized
inertia tensor diagonal (inertia tensors are typically represented by a 3x3
matrix). We multiply this by the mass to get the full inertia. Also, this
formula expects the collider to already be scaled.

### Capsule Colliders

The center of mass of a capsule is also very easy. It is simply the midpoint of
its interior segment.

For the angular expansion, Unity represents it as half the length of the
segment.

The inertia tensor is where things get interesting. Unity calculates the inertia
tensor diagonal with the segment realigned to the y-axis. That means it keeps a
quaternion involved, and we probably aren’t accounting for this quaternion
anywhere.

Turns out, this quaternion is supposed to be factored into our
`centerOfMassWorldTransformA/B` in our `BuildJacobian()` method. We’ll have to
rename that appropriately to make the clearer. I picked
`inertialPoseWorldTransformA/B`.

That also means we’ll need to return both the inertia tensor diagonal and its
orientation. So we’ll define a struct for that:

```csharp
public struct LocalInertiaTensorDiagonal
{
    public quaternion tensorOrientation;
    public float3     inertiaDiagonal;
}
```

The actual math for computing this for capsules is a little intense. And I
tweaked it a bit compared to Unity Physics to more accurately cover the
degenerate case. We’ll trust that Burst will know how to chew through this and
make this fast enough for real-time.

```csharp
case ColliderType.Capsule:
{
    float3 axis     = collider.m_capsule.pointB - collider.m_capsule.pointA;
    float  lengthSq = math.lengthsq(axis);
    float  length   = math.sqrt(lengthSq);
    if (lengthSq == 0f || !math.isfinite(length))
    {
        new LocalInertiaTensorDiagonal
        {
            inertiaDiagonal   = (2f / 5f) * collider.m_capsule.radius * collider.m_capsule.radius,
            tensorOrientation = quaternion.identity
        };
    }

    float radius   = collider.m_capsule.radius;
    float radiusSq = radius * radius;

    float cylinderMassPart  = math.PI * length * radiusSq;
    float sphereMassPart    = math.PI * (4f / 3f) * radiusSq * radius;
    float totalMass         = cylinderMassPart + sphereMassPart;
    cylinderMassPart       /= totalMass;
    sphereMassPart         /= totalMass;

    float onAxisInertia  = (cylinderMassPart / 2f + sphereMassPart * 2f / 5f) * radiusSq;
    float offAxisInertia = cylinderMassPart * (radiusSq / 4f + lengthSq / 12f) +
                            sphereMassPart * (radiusSq * 2f / 5f + radius * length * 3f / 8f + lengthSq / 4f);

    return new LocalInertiaTensorDiagonal
    {
        inertiaDiagonal   = new float3(offAxisInertia, onAxisInertia, offAxisInertia),
        tensorOrientation = mathex.FromToRotation(new float3(0f, 1f, 0f), math.normalize(axis))
    };
}
```

### Box Colliders

Believe it or not, box colliders may actually be simpler than capsule colliders.
Just like spheres, their center of mass is defined at their own center.

For angular expansion, it is the length of the `halfSize`. Note that Unity
Physics uses full widths in its box collider representation, so the formulas
here will be adjusted accordingly.

As for the inertia tensor, the local-space box is already axis-aligned, so we
have an identity orientation. And the formula is quite simple.

```csharp
case ColliderType.Box:
{
    var halfSq = collider.m_box.halfSize * collider.m_box.halfSize;
    float3 tensorNum = new float3(halfSq.y + halfSq.z, halfSq.x + halfSq.z, halfSq.x + halfSq.y);
    return new LocalInertiaTensorDiagonal
    {
        inertiaDiagonal = tensorNum / 3f,
        tensorOrientation = quaternion.identity
    };
}
```

### Triangle Colliders

Triangle colliders are where Unity starts to get really hacky with their
implementation. However, the center of mass is still trivial, and is just the
average of the vertices.

The angular expansion is just the distance from the center to the farthest away
vertex.

As for the inertia tensor, Unity decided that they don’t want to calculate it
because it is apparently too expensive. So instead they compute the inertia
tensor of the bounding sphere. Good thing we are putting these utilities inside
the `UnitySim` static class!

### Convex Colliders

Now things are about to get real interesting. Unity’s `ConvexHullBuilder`
provides a `hullMassProperties` that gives us some useful info. One piece is the
center of mass, so once again that remains easy. We just store that in the blob.
But we’ll need to multiply it by the collider’s scale.

For angular expansion, Unity computes the length of the furthest vertex from the
center of mass. Unfortunately, this isn’t a value we can cache because of
stretch. Instead, we’ll use the distance from the center of mass to the furthest
AABB corner, which will act as a conservative approximation. Angular expansion
is meant to be a conservative approximation value anyways, so this shouldn’t
break anything. Here’s what that looks like:

```csharp
case ColliderType.Convex:
{
    ref var blob = ref collider.m_convex.convexColliderBlob.Value;
    var aabb = blob.localAabb;
    aabb.min *= collider.m_convex.scale;
    aabb.max *= collider.m_convex.scale;
    var centerOfMass = blob.centerOfMass;
    Physics.GetCenterExtents(aabb, out var aabbCenter, out var  aabbExtents);
    var delta = centerOfMass - aabbCenter;
    var extremePoint = math.select(aabbExtents, -aabbExtents, delta >= 0f);
    return math.distance(extremePoint, delta);
}
```

Now, the inertia tensor is where things get tricky. We can get an inertia tensor
from the `ConvexHullBuilder`. The problem is that it is the full 3x3 matrix.
Unity Physics takes this value and computes a diagonal approximation using an
iterative process with 10 iterations. That’s not cheap.

There probably is a way to update an inertial tensor diagonal and orientation
given a non-uniform scale, but Unity Physics doesn’t have that fast method. For
now, we’ll bite the cost and cache the diagonal and orientation of identity
scale as a common fast path.

```csharp
case ColliderType.Convex:
{
    ref var blob = ref collider.m_convex.convexColliderBlob.Value;
    var scale = collider.m_convex.scale;
    if (scale.Equals(new float3(1f, 1f, 1f)))
    {
        // Fast path
        return new LocalInertiaTensorDiagonal
        {
            inertiaDiagonal   = blob.unscaledInertiaTensorDiagonal,
            tensorOrientation = blob.unscaledInertiaTensorOrientation
        };
    }

    // Todo: Is there a faster way to do this when we already have the unscaled orientation and diagonal?
    var scaledTensor = blob.inertiaTensor;
    scaledTensor.c0.x *= scale.x;
    scaledTensor.c1.y *= scale.y;
    scaledTensor.c2.z *= scale.z;
    mathex.DiagonalizeSymmetricApproximation(scaledTensor, out var orientation, out var diagonal);
    return new LocalInertiaTensorDiagonal
    {
        inertiaDiagonal   = diagonal,
        tensorOrientation = new quaternion(orientation)
    };
}
```

If you know a faster method, please let me know!

### TriMesh Colliders

Given how many corners Unity cut with just the triangle collider, how much do
you expect out of triangle mesh?

Yeah… no.

Unity pretends the mesh’s AABB is a box collider and computes the values for
that instead. We’ll follow suit for now. But I bet someone out there will want
to override this behavior with some proper values computed from another library
or something.

### Compound Colliders

You would think that Unity would cut corners here too. But nope. They take
compound colliders seriously. Fortunately, the runtime characteristics will be
identical to convex colliders in how we handle the various properties. The
heaviest calculations will be contained within blob baking. But unlike with the
magic `ConvexHullBuilder`, we need to calculate these combined cached values
ourselves.

I won’t share the code, because it is a little lengthy and tedious. But it
involved keeping a cache of volumes of each child, their inertia tensor, and
their center of mass, and then using all of those to reorient and scale a final
inertia tensor. I also needed to add this `ToMatrix()` method to the
`LocalInertiaTensorDiagonal`.

```csharp
public float3x3 ToMatrix()
{
    var r  = new float3x3(tensorOrientation);
    var r2 = new float3x3(inertiaDiagonal.x * r.c0, inertiaDiagonal.y * r.c1, inertiaDiagonal.z * r.c2);
    return math.mul(r2, math.transpose(r));
}
```

### Priming Our Inputs

Now that we have all of these properties, we need to be able to convert them
into the form that our solver desires.

```csharp
public static void ConvertToWorldMassInertia(in TransformQvvs worldTransform,
                                                in LocalInertiaTensorDiagonal localInertiaTensorDiagonal,
                                                float3 localCenterOfMass,
                                                float inverseMass,
                                                out Mass massOut,
                                                out RigidTransform inertialPoseWorldTransform)
{
    inertialPoseWorldTransform = new RigidTransform(qvvs.TransformRotation(in worldTransform, localInertiaTensorDiagonal.tensorOrientation),
                                                    qvvs.TransformPoint(in worldTransform, localCenterOfMass));
    massOut = new Mass
    {
        inverseMass    = inverseMass,
        inverseInertia = math.rcp(localInertiaTensorDiagonal.inertiaDiagonal) * inverseMass / (worldTransform.scale * worldTransform.scale)
    };
}
```

Little conversion utilities like this simply exist as examples. Individuals
looking for maximum performance will won’t to write their own implementations
that take advantage of as many assumptions as they can.

## Motion Expansion

Unity Physics collision detection is continuous, using a technique called
“speculative contacts”. That means that Unity tries to anticipate body pairs
that will collide during the time step simply by looking at the initial
transforms and velocities. Because of this, the AABBs used in the broad-phase
get expanded. And the max distance value used in distance queries will be
greater than 0. Psyshock is able to accept these modified inputs. We just need
to calculate them.

Unity Physics represents the parameters of motion expansion as uniform and
directional parts. And for each pair, it combines these parameters between
bodies to produce the max distance value for a distance query. We can construct
these parameters for a single body from the velocity, timestep, and angular
expansion factor. That looks like this:

```csharp
public struct MotionExpansion
{
    float4 uniformXlinearYzw;

    public MotionExpansion(in Velocity velocity, float deltaTime, float angularExpansionFactor)
    {
        var linear = velocity.linear * deltaTime;
        // math.length(AngularVelocity) * timeStep is conservative approximation of sin((math.length(AngularVelocity) * timeStep)
        var uniform = 0.05f + math.min(math.length(velocity.angular) * deltaTime * angularExpansionFactor, angularExpansionFactor);
        uniformXlinearYzw = new float4(uniform, linear);
    }
}
```

There’s that little `0.05f` in there. What is that about?

Well, that value actually comes from Unity Physics
`CollisionWorld.CollisionTolerance`. Half of that is applied to each motion
expansion’s `Aabb`, but only if the body is dynamic. However, when applying the
distance query, the full value is added to the distance independently. This
causes a mismatch between static and dynamic bodies. This might be a bug in
Unity Physics? Regardless, we can simplify the math involved by including it
inline in the `MotionExpansion` type, as Unity hardcodes the value to `0.1f`
anyways.

It is also worth noting that Unity pre-applies gravity to the velocity passed
in. But what it passes in is a temporary copy, because real gravity is applied
later in integration.

Expanding and `Aabb` instance then looks like this. An AABB is always expanded
by at least `0.05f` even if it isn’t moving.

```csharp
public Aabb ExpandAabb(Aabb aabb)
{
    var linear = uniformXlinearYzw.yzw;
    aabb.min = math.min(aabb.min, aabb.min + linear) - uniformXlinearYzw.x;
    aabb.max = math.max(aabb.max, aabb.max + linear) + uniformXlinearYzw.x;
    return aabb;
}
```

Next, we need to compute the max distance values. We have two variants, one when
our dynamic body is paired with a static body, and the other when two dynamic
bodies are involved.

```csharp
public static float GetMaxDistance(in MotionExpansion motionExpansion)
{
    return math.length(motionExpansion.uniformXlinearYzw.yzw) + motionExpansion.uniformXlinearYzw.x + 0.05f;
}

public static float GetMaxDistance(in MotionExpansion motionExpansionA, in MotionExpansion motionExpansionB)
{
    var tempB = motionExpansionB.uniformXlinearYzw;
    tempB.x = -tempB.x;
    var tempCombined = motionExpansionA.uniformXlinearYzw - tempB;
    return math.length(tempCombined.yzw) + tempCombined.x;
}
```

And just like that, we have all the pieces we need to go from simple user inputs
all the way through the solver. In addition to the expected things like
transform and collider shapes, all the user needs to provide is the mass,
gravity, and coefficient of restitution. Everything else has some kind of
utility to calculate its value.

## We’re Missing Something

Where’s friction?

Shouldn’t that also be something the user provides?

Yes. And it is supposed to go into our `ContactJacobianBodyParameters`, but I
forgot to set it. So we’ll need to add that argument in our `BuildJacobian()`
method.

And I now realize that I also forgot to set the contact normal and friction
directions. I’m glad I caught that now.

Anyways, where were we? Right. We have all these values ready to go through our
solver. All that’s left is to actually move our bodies. And that’s where
integration comes in.

## Integration

Unity Physics splits the integration up into a bunch of helper methods. However,
I found myself jumping through code a lot to track down everything that was
happening. Instead, I packed everything inside a single method. This is what
that looks like:

```csharp
public static void Integrate(ref RigidTransform inertialPoseWorldTransform, ref Velocity velocity, float linearDamping, float angularDamping, float deltaTime)
{
    inertialPoseWorldTransform.pos += velocity.linear * deltaTime;
    var halfDeltaAngle              = velocity.angular * 0.5f * deltaTime;
    var dq                          = new quaternion(new float4(halfDeltaAngle, 1f));
    inertialPoseWorldTransform.rot  = math.normalize(math.mul(inertialPoseWorldTransform.rot, dq));

    var dampFactors   = math.clamp(1f - new float2(linearDamping, angularDamping) * deltaTime, 0f, 1f);
    velocity.linear  *= dampFactors.x;
    velocity.angular *= dampFactors.y;
}
```

The first line is the simple position integration with velocity. The next three
lines are integrating the rotation. It is your typical quaternion weirdness.
Don’t stare at it too long.

Then the velocities get updated with damping factors. Those are really the
“drag” values in authoring.

### Where’s Gravity?

Earlier with motion expansion, I mentioned Unity was computing a temporary
velocity that included gravity, because it was going to apply the real gravity
later. Well, we’re at the “later” and there’s no gravity in sight.

In classic Unity fashion, gravity is being updated right after motion expansion.
Why Unity can’t do the two together? I have no idea. But we should update our
comment accordingly. In Psyshock, it makes way too much sense to update it only
once.

### Writing Back

Our integration updates the `inertialPoseWorldTransform` which we know is based
on the body’s center of mass. But that’s different from our entity’s transform.
We need a way to update our entity’s `WorldTransform` given the updated
`inertialPoseWorldTransform`.

We can either calculate this by comparing the delta of
`inertialPoseWorldTransform` and the entity’s old `WorldTransform`, or we can
bring in the local center of mass and inertia tensor orientation. Here’s the
implementation for both:

```csharp
public static TransformQvvs ApplyInertialPoseWorldTransformDeltaToWorldTransform(in TransformQvvs oldWorldTransform,
                                                                                    in RigidTransform oldInertialPoseWorldTransform,
                                                                                    in RigidTransform newInertialPoseWorldTransform)
{
    var oldTransform = new RigidTransform(oldWorldTransform.rotation, oldWorldTransform.position);
    // oldInertialPoseWorldTransform = oldWorldTransform * localInertial
    // newInertialPoseWorldTransfrom = newWorldTransform * localInertial
    // inverseOldWorldTransform * oldInertialWorldTransform = inverseOldWorldTransform * oldWorldTransform * localInertial
    // inverseOldWorldTransform * oldInertialWorldTransform = localInertial
    // newInertialPoseWorldTransform * inverseLocalInertial = newWorldTransform * localInertial * inverseLocalInertial
    // newInertialPoseWorldTransform * inverseLocalInertial = newWorldTransform
    // newInertialPoseWorldTransform * inverse(inverseOldWorldTransform * oldInertialWorldTransform) = newWorldTransform
    // newInertialPoseWorldTransform * inverseOldInertialWorldTransform * oldWorldTransform = newWorldTransform
    var newTransform = math.mul(newInertialPoseWorldTransform, math.mul(math.inverse(oldInertialPoseWorldTransform), oldTransform));
    return new TransformQvvs
    {
        position   = newTransform.pos,
        rotation   = newTransform.rot,
        scale      = oldWorldTransform.scale,
        stretch    = oldWorldTransform.stretch,
        worldIndex = oldWorldTransform.worldIndex
    };
}

public static void ApplyWorldTransformFromInertialPoses(ref TransformQvvs worldTransform,
                                                        in RigidTransform inertialPoseWorldTransform,
                                                        quaternion localTensorOrientation,
                                                        float3 localCenterOfMassUnscaled)
{
    var localInertial       = new RigidTransform(localTensorOrientation, localCenterOfMassUnscaled * worldTransform.stretch * worldTransform.scale);
    var newWorldTransform   = math.mul(inertialPoseWorldTransform, math.inverse(localInertial));
    worldTransform.position = newWorldTransform.pos;
    worldTransform.rotation = newWorldTransform.rot;
}
```

Which you decide to use will largely depend on which data you have available to
you at the time of the call.

## What’s Next

At this point, we have all the individual pieces we need to start simulating. It
certainly won’t be the most stable of simulations since we don’t have the
stabilizer logic. And it won’t be the most feature-rich simulation since we
don’t have joints or motors yet. But it will be a simulation. And if the
simulation works, we can start adding all those other pieces.

But before that, we need to put all these pieces together. Our simulation can be
broken up into four phases. First, we apply all our forces like gravity to our
velocity and build our collision layers. Next, we run out collision detection
and generate our contacts and constraints (AKA Jacobians according to Unity).
After that, we run our solver. And lastly, we integrate our velocities into our
positions and rotations and apply them back to our entities.

Next time, we’ll build an example physics engine inside ECS with all these
pieces, and then discover how close we were to getting the simulation pieces
right.

Thanks for reading!
