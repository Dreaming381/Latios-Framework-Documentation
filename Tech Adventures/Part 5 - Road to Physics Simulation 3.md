# Tech Adventures: Road to Physics Simulation Part 3

Something I’ve been learning about physics simulation lately is that rigid body
physics is all about making things rotate when they touch. Seriously, if all
they had to do was move, we’d be done with this journey already. Technically,
that’s the world of soft-body physics which I personally find way more
fascinating. But that also requires good art to show it off, and so far no art
has arrived on my GitHub doorstep with a “please use” tag. While I could go out
and find art, I’d prefer to not be one of *those* dragons. Besides, I have
unpopular art tastes anyways.

Anyways, rigid body physics is all about making things rotate and touch, and in
this part 3 of our adventure, we’re going to figure out the “touch” part.

Problem, what is a “touch”?

It sounds like a stupid question, but believe it or not, this is something
physics engines argue with each other about. And every engine thinks their touch
is the best touch for performance, accuracy, or whatever other metric they want
to brag about. Psyshock’s architecture makes such arguments meaningless, as it
can include and mix and match whichever ones we decide to implement. So we’re
just going to pick one to start off with, and probably add more later on. Let’s
start with Unity Physics, since people know that one and we have the full
DOTS-compatible source code for it.

## Unity Contact Representation

Before we can generate any contacts, we need to figure out what a contact looks
like for Unity. Remember, we are dealing with non-realistic situations where
objects may be overlapping. Consequently, contacts points come in pairs, one on
each collider primitive. Unity Physics compresses this by only storing one of
these contact positions, and then storing the distance to the other contact.

But which direction do we need to travel to recover the opposite contact?

Unity defines all contacts such that they all share the same direction, which it
refers to as the **contact normal**. This is *not* the surface normal, but
rather is the normalized direction between the hitpoints from a
`DistanceBetween`. And if that fails, then a guess the surface normal is a good
enough fallback.

Lastly, how many contacts does Unity use for two primitives? Up to 32.

And so, we arrive at our output data structure. It is a little different from
what Unity uses internally, as I am experimenting with a different API and
memory layout. But all the important pieces are there.

```csharp
public unsafe struct ContactsBetweenResult
{
    public float3 contactNormal;
    public int contactCount;
    public fixed float contactsData[128];

    public ref Contact this[int index]
    {
        get
        {
            CheckInRange(index);
            fixed(void* ptr = contactsData)
                return ref ((Contact*)ptr)[index];
        }
    }

    public void Add(Contact contact)
    {
        CheckCapacityBeforeAdd();
        this[contactCount] = contact;
        contactCount++;
    }

    public void Remove(int index)
    {
        CheckInRange(index);
        this[index] = this[contactCount - 1];
        contactCount--;
    }

    public struct Contact
    {
        public float4 contactData;
        public float3 location
        {
            get => contactData.xyz;
            set => contactData.xyz = value;
        }
        public float distance
        {
            get => contactData.w;
            set => contactData.w = value;
        }
    }

    [Conditional("ENABLE_UNITY_COLLECTIONS_CHECKS")]
    void CheckCapacityBeforeAdd()
    {
        if (contactCount >= 32)
            throw new System.InvalidOperationException("Cannot add more than 32 contacts.");
    }

    [Conditional("ENABLE_UNITY_COLLECTIONS_CHECKS")]
    void CheckInRange(int index)
    {
        if (index < 0 || index >= contactCount)
            throw new System.ArgumentOutOfRangeException($"Contact index {index} is out of range of [0, {contactCount})");
    }
}
```

There’s quite the nice API coming along here. However, there is one big
oversight with this design that needs to be fixed during our adventure. Do these
contacts correspond to the first collider in the pair, or the second?

Unfortunately, Unity doesn’t specify anywhere in their code or their docs.
Hopefully we’ll figure it out along the way so that I can rename things to be
specific. But for now, it is time to enter the 15-scenario gauntlet once again.

## Sphere Vs Sphere

With a sphere vs sphere scenario, we only expect a single contact pair. But
which point do we store? It could be on A, or it could be on B, and I even know
engines that plop it right in the middle. Let’s see what Unity Physics does:

```csharp
DistanceQueries.Result convexDistance = DistanceQueries.SphereSphere(sphereA, sphereB, aFromB);
if (convexDistance.Distance < maxDistance)
{
    manifold = new Manifold(convexDistance, worldFromA);
}
```

So what we have now is Unity’s version of `DistanceBetween`, and then that being
dropped directly into the constructor of a `Manifold`, which is Unity’s
equivalent of our `ContactsBetweenResult`. I guess we have to look at that
constructor to see what’s really happening.

```csharp
// Create a single point manifold from a distance query result
public Manifold(DistanceQueries.Result convexDistance, MTransform worldFromA)
{
    NumContacts = 1;
    Normal = math.mul(worldFromA.Rotation, convexDistance.NormalInA);
    this[0] = new ContactPoint
    {
        Distance = convexDistance.Distance,
        Position = Mul(worldFromA, convexDistance.PositionOnBinA)
    };
}
```

Aha!

The contact is on B!

I’ve done some renames and wrote this convenient `Add()` overload which shows
our discovery off:

```csharp
public void Add(float3 locationOnB, float distanceToA)
{
    Add(new ContactOnB { location =  locationOnB, distanceToA = distanceToA });
}
```

And thus, our sphere vs sphere contact algorithm looks like this:

```csharp
public static UnitySim.ContactsBetweenResult UnityContactsBetween(in SphereCollider sphereA,
                                                                    in RigidTransform aTransform,
                                                                    in SphereCollider sphereB,
                                                                    in RigidTransform bTransform,
                                                                    in ColliderDistanceResult distanceResult)
{
    UnitySim.ContactsBetweenResult result = default;
    result.contactNormal                  = distanceResult.normalB;
    result.Add(distanceResult.hitpointB, distanceResult.distance);
    return result;
}
```

We use `normalB` as the contact normal because for sphere vs sphere it always
points towards A, plus it already handles the cases where the hitpoints are
equal. As for the arguments, we assume the user has first called a
`DistanceBetween` method and we are working with the results. In sphere vs
sphere, most of the arguments are unused. But we will need those for the other
scenarios. I include them here to make the dispatch code work, but Burst will
probably inline this method and optimize them out.

Anyways, that’s one case down (admittedly, the easiest case). 14 more to go!

## Sphere vs Capsule

For capsule vs sphere, we start to enter the scenario where it matters a lot
more who is A and who is B. Unity always assumes that the Capsule is A in this
pair. And then if that’s not the case, it flips the result by calculating the
contact points on A and flipping the contact normal.

Perhaps by coincidence, Psyshock also uses this Capsule-is-A convention in
`DistanceBetween`, so there isn’t much else to account for. Contact generation
is pretty much identical to the sphere vs sphere case. We’ll deal with the
dispatchers after we’re done with all 15 cases.

```csharp
public static UnitySim.ContactsBetweenResult UnityContactsBetween(in CapsuleCollider capsule,
                                                                    in RigidTransform capsuleTransform,
                                                                    in SphereCollider sphere,
                                                                    in RigidTransform sphereTransform,
                                                                    in ColliderDistanceResult distanceResult)
{
    UnitySim.ContactsBetweenResult result = default;
    result.contactNormal                  = distanceResult.normalB;
    result.Add(distanceResult.hitpointB, distanceResult.distance);
    return result;
}
```

## Capsule vs Capsule

This scenario introduces a new concept. Both collider types have multiple
features, meaning we can have more than one contact pair. So let’s see how Unity
handles it…

```csharp
// TODO: Should produce a multi-point manifold
DistanceQueries.Result convexDistance = DistanceQueries.CapsuleCapsule(capsuleA, capsuleB, aFromB);
if (convexDistance.Distance < maxDistance)
{
    manifold = new Manifold(convexDistance, worldFromA);
}
```

Huh.

I guess Unity doesn’t handle it at all!

It is weird, because there’s actually some good references online for how to
handle this case. But regardless, our goal is to replicate what Unity does, so
we are going to take the easy way out here too.

```csharp
public static UnitySim.ContactsBetweenResult UnityContactsBetween(in CapsuleCollider capsuleA,
                                                                    in RigidTransform aTransform,
                                                                    in CapsuleCollider capsuleB,
                                                                    in RigidTransform bTransform,
                                                                    in ColliderDistanceResult distanceResult)
{
    // As of Unity Physics 1.0.14, only a single contact is generated for this case,
    // and there is a todo for making it multi-contact.
    UnitySim.ContactsBetweenResult result = default;
    result.contactNormal                  = distanceResult.normalB;
    result.Add(distanceResult.hitpointB, distanceResult.distance);
    return result;
}
```

## Sphere vs Box

Back to spheres again. Can you guess what the solution is?

```csharp
public static UnitySim.ContactsBetweenResult UnityContactsBetween(in BoxCollider box,
                                                                    in RigidTransform boxTransform,
                                                                    in SphereCollider sphere,
                                                                    in RigidTransform sphereTransform,
                                                                    in ColliderDistanceResult distanceResult)
{
    UnitySim.ContactsBetweenResult result = default;
    result.contactNormal                  = distanceResult.normalB;
    result.Add(distanceResult.hitpointB, distanceResult.distance);
    return result;
}
```

This is starting to become a pattern-recognition test for Burst.

## Capsule vs Box

It turns out that Unity’s laziness for capsule vs capsule was a warning of what
was to come, because there’s not even a specialization for capsule vs box. Unity
uses a general-purpose convex vs convex algorithm, and it is one we will need to
dissect in order to extract the critical pieces and maybe come up with some
optimizations along the way.

Fortunately, the algorithm has several critical branches based on each
collider’s number of faces and vertices. If we assume our box is A with zero
radius and our capsule is B, we can boil the entire method down to the
following:

```csharp
ConvexConvexDistanceQueries.Result result = ConvexConvexDistanceQueries.ConvexConvex(
    vertexPtrA, hullA.NumVertices, vertexPtrB, hullB.NumVertices, aFromB, ConvexConvexDistanceQueries.PenetrationHandling.Exact3D);

float sumRadii = convexRadiusB;
if (result.ClosestPoints.Distance < maxDistance + sumRadii)
{
    float3 normal = result.ClosestPoints.NormalInA;

    manifold = new Manifold
    {
        Normal = math.mul(worldFromA.Rotation, normal)
    };
    int faceIndexA = hullA.GetSupportingFace(-normal, result.SimplexVertexA(0), planesA);
    if (FaceEdge(vertexPtrA, vertexPtrB, faceIndexA, worldFromA, aFromB, normal, result.ClosestPoints.Distance, ref manifold,
        planesA, ref hullA, convexRadiusA, convexRadiusB, aNegativeScaled))
    {
        return;
    }
    // Either one of the shapes is a sphere, or both of the closest features are nearly perpendicular to the contact normal,
    // or FaceEdge() missed the closest point due to numerical error.  In these cases, add the closest point directly to the manifold.
    if (manifold.NumContacts < Manifold.k_MaxNumContacts)
    {
        DistanceQueries.Result convexDistance = result.ClosestPoints;
        manifold[manifold.NumContacts++] = new ContactPoint
        {
            Position = Mul(worldFromA, convexDistance.PositionOnAinA) - manifold.Normal * (convexDistance.Distance - convexRadiusB),
            Distance = convexDistance.Distance - sumRadii
        };
    }
}
else
{
    manifold = new Manifold();
}
```

We know how to deal with everything right up to `faceIndexA`, and then we have
some new methods we need to make sense of. Unity uses GJK collision detection
for capsule vs box, so it creates a simplex. In `SimplexVertexA(0)`, it is
asking for the vertex on the box that is part of the first simplex point.
There’s a chance we may need to make sense of how the first simplex point is
arrived to, but let’s see what this magic `GetSupportingFace()` method does.

Yeah… there’s actually a lot to this method that we will dig into later, but I
am going to provide some context clues that will be sufficient for handling box
colliders.

In a simplex in GJK, there are up to four pairs of vertices, but only up to
three unique vertices can be provided by each collider, meaning that there may
be duplicates. How many vertices each provides specifies the type of closest
feature for that collider. The goal of `GetSupportingFace()` is to find a face
that either is or contains the closest feature with the normal most aligned to
the passed in direction. That direction is the contact normal, which for convex
colliders should mean that the face with the normal most aligned with that
direction should always contain the closest feature.

For a box, that’s just the face with the normal that matches the dominant axis
of the direction and sign of that axis in the box’s local space. This is pretty
easy to calculate, but we still need to figure out what form we need to
represent this face in for the next step, which is the `FaceEdge()` method.

The `FaceEdge()` method is fallible, and when it fails, the generalized convex
contact manifold algorithm falls back onto generating a single contact point the
same way we’ve been doing it for all the pair combinations so far. The first
failure check is when the face normal is nearly perpendicular to the closest
points direction – specifically, if the dot product between the two is less than
`0.05f`. Because we are picking the box face with the largest axis magnitude of
the input direction, it is impossible for the normal to be that close to
perpendicular. There’s always a face aligned within 45 degrees. We can skip this
check.

The only two other failure cases are if we run out of contacts (can’t happen
here) or all the found contact points on B (the capsule) are outside the search
distance. Initially, the latter detail appears to be one we didn’t account for
in our internal API design. However, the value comes from the result distance of
our distance finding method, and not our original search distance.

The remaining algorithm treats the capsule edges as rays, and raycasts against
the edge planes of our chosen face with the planes facing outwards. These edge
planes are oriented around the contact normal and not the face normal. The
raycast captures the first “enter” hit where the ray passes a plane from the
outside, and the first “exit” hit, where the ray passes the plane from the
inside. These hitpoints clamp the capsule segment to what is projected onto the
plane along the contact normal, and these become the contact points. This
process of shortening projected edges by using raycasts against the edge planes
is often referred to as “clipping” and is the fundamental basis of all these
multi-contact algorithms.

As long as the ray does not exit an edge plane before entering one, the two
contact points are always added. However, the method may still report a failure
if neither of the contact points also contain a point nearly the same distance
away as the closest point (0.0001 tolerance). Additionally, if the ray does exit
an edge plane before entering one, the segment never projects onto the face and
the method reports a failure.

Alright! Let’s put this algorithm together!

First, we need the box plane and edge planes. Because we know the exact anatomy
of our box, we can identify the face index and then use a switch-case to
populate the values we care about. We’ll actually need this operation for
multiple pair types, so I made it its own method.

```csharp
internal static void BestFacePlanesAndVertices(in BoxCollider box,
                                                float3 localDirectionToAlign,
                                                out simdFloat3 edgePlaneOutwardNormals,
                                                out float4 edgePlaneDistances,
                                                out Plane plane,
                                                out simdFloat3 vertices)
{
    var axisMagnitudes = math.abs(localDirectionToAlign);
    var bestAxis       = math.cmax(axisMagnitudes) == axisMagnitudes;
    // Prioritize y first, then z, then x if multiple distances perfectly match.
    // Todo: Should this be configurabe?
    bestAxis.xz               &= !bestAxis.y;
    bestAxis.x                &= !bestAxis.z;
    bool   bestAxisIsNegative  = math.any(bestAxis & (localDirectionToAlign < 0f));
    var    faceIndex           = math.tzcnt(math.bitmask(new bool4(bestAxis, false))) + math.select(0, 3, bestAxisIsNegative);
    float4 ones                = 1f;
    float4 firstComponent      = new float4(-1f, 1f, 1f, -1f);
    float4 secondCompPos       = new float4(1f, 1f, -1f, -1f);  // CW so that edge X plane_normal => outward
    switch (faceIndex)
    {
        case 0:  // positive X
            plane    = new Plane(new float3(1f, 0f, 0f), box.halfSize.x);
            vertices = new simdFloat3(ones, firstComponent, secondCompPos);
            break;
        case 1:  // positive Y
            plane    = new Plane(new float3(0f, 1f, 0f), box.halfSize.y);
            vertices = new simdFloat3(firstComponent, ones, secondCompPos);
            break;
        case 2:  // positive Z
            plane    = new Plane(new float3(0f, 0f, 1f), box.halfSize.z);
            vertices = new simdFloat3(firstComponent, secondCompPos, ones);
            break;
        case 3:  // negative X
            plane    = new Plane(new float3(-1f, 0f, 0f), -box.halfSize.x);
            vertices = new simdFloat3(-ones, firstComponent, -secondCompPos);
            break;
        case 4:  // negative Y
            plane    = new Plane(new float3(0f, -1f, 0f), -box.halfSize.y);
            vertices = new simdFloat3(firstComponent, -ones, -secondCompPos);
            break;
        case 5:  // negative Z
            plane    = new Plane(new float3(0f, 0f, -1f), -box.halfSize.z);
            vertices = new simdFloat3(firstComponent, -secondCompPos, -ones);
            break;
        default:  // Should not happen
            plane    = default;
            vertices = default;
            break;
    }
    vertices                *= box.halfSize;
    edgePlaneOutwardNormals  = simd.cross(vertices.bcda - vertices, localDirectionToAlign);  // These normals are perpendicular to the contact normal, not the plane.
    edgePlaneDistances       = simd.dot(edgePlaneOutwardNormals, vertices.bcda);
}
```

The next step was to perform our raycast against the four edge planes. I went
full SIMD with this. Though this could maybe be optimized further by reducing
the vectors to scalars along the axes in question.

```csharp
var boxLocalContactNormal = math.InverseRotateFast(boxTransform.rot, -distanceResult.normalB);
PointRayBox.BestFacePlanesAndVertices(in box, boxLocalContactNormal, out var edgePlaneNormals, out var edgePlaneDistances, out var plane, out _);

var  bInATransform     = math.mul(math.inverse(boxTransform), capsuleTransform);
var  rayStart          = math.transform(bInATransform, capsule.pointA);
var  rayDisplacement   = math.transform(bInATransform, capsule.pointB) - rayStart;
var  rayRelativeStarts = simd.dot(rayStart, edgePlaneNormals) - edgePlaneDistances;
var  relativeDiffs     = simd.dot(rayDisplacement, edgePlaneNormals);
var  rayRelativeEnds   = rayRelativeStarts + relativeDiffs;
var  rayFractions      = math.select(-rayRelativeStarts / relativeDiffs, float4.zero, relativeDiffs == float4.zero);
var  startsInside      = rayRelativeStarts <= 0f;
var  endsInside        = rayRelativeEnds <= 0f;
var  projectsOnFace    = startsInside | endsInside;
var  enterFractions    = math.select(float4.zero, rayFractions, !startsInside & rayFractions > float4.zero);
var  exitFractions     = math.select(1f, rayFractions, !endsInside & rayFractions < 1f);
var  fractionA         = math.cmax(enterFractions);
var  fractionB         = math.cmin(exitFractions);
```

Lastly, we need to compute the clipped segment points and distances to the box
and generate our contacts. Unity does some weird math where they try to express
the distances relative to the ray fractions. However, I came up with a more
intuitive set of expressions that seem to require the same amount of operations.
Note that we need to scale our distance such that it is the distance to contact
along the contact normal, and not along the face normal.

```csharp
bool needsClosestPoint = true;
if (math.all(projectsOnFace) && fractionA < fractionB)
{
    // Add the two contacts from the possibly clipped segment
    var distanceScalarAlongContactNormal = math.rcp(math.dot(boxLocalContactNormal, plane.normal));
    var clippedSegmentA = rayStart + fractionA * rayDisplacement;
    var clippedSegmentB = rayStart + fractionB * rayDisplacement;
    var aDistance       = mathex.SignedDistance(plane, clippedSegmentA) * distanceScalarAlongContactNormal;
    var bDistance       = mathex.SignedDistance(plane, clippedSegmentB) * distanceScalarAlongContactNormal;
    result.Add(math.transform(boxTransform, clippedSegmentA), aDistance - capsule.radius);
    result.Add(math.transform(boxTransform, clippedSegmentB), bDistance - capsule.radius);
    needsClosestPoint = math.min(aDistance, bDistance) > distanceResult.distance + 1e-4f;  // Magic constant comes from Unity Physics
}

if (needsClosestPoint)
    result.Add(distanceResult.hitpointB, distanceResult.distance);
return result;
```

And that’s it. That’s our first multi-contact algorithm down.

## Box vs Box

Up until this point, every contact generation algorithm in Unity Physics started
by performing a collider distance query. Box vs box is where that pattern stops.
This method starts by performing custom distance queries using SAT. The method
only finds the contact normal (which only matches the contact normal by our
previous definitions if the boxes are intersecting) and the distance via SAT,
and does not compute the closest points immediately. It then tries its
`FaceFace()` algorithm for generating contacts between two support faces which
are found the same way as in Capsule vs Box. If it fails to find contacts
roughly within the SAT distance, the algorithm reverts to the generalized
contact algorithm where it uses GJK to find the closest points and then
effectively repeats trying to build contacts with `FaceFace()`.

As for the `FaceFace()` method, it takes all the edges around the face of A and
clips them against the face of B, deduplicating where needed (it discards the
contact of `fractionB` if `fractionB` is `1f`). Then it takes any vertices from
the face of B and if they project onto the face of A, it adds those as contact
points as well. Curiously, it never clips the edges of the face of B onto the
face of A. There’s a little bit of asymmetry. But if you think about it more
carefully, you’ll realize this is deduplicating the intersections of projected
edges between the two faces.

We can heavily rely on some of the techniques we developed for Capsule vs Box
here. But there’s still one big unsolved problem. Unlike rounded shapes where
the surface normal was equivalent to the contact normal, pairs of boxes don’t
have such trivial contact normal calculations. We’ll have to derive them from
the feature codes.

If any of the feature codes is a face, we can assume the other feature code is
supposed to be a point. Even if it isn’t, we’ll pretend it is. In such cases,
the contact normal will be based on the contact normal of the face feature code.
In the case of the face being on B, we need to convert it to A space and flip
the signs, as this normal should mostly represent the direction from the
hitpoint on A to the hitpoint on B when multiplied by the distance (negative
distance means opposite direction). Note, this is backwards to how we store the
contact normal in the `ContactsBetweenResult`.

```csharp
case 2: // A point and B face
case 6: // A edge and B face
{
    var faceIndex = distanceResult.featureCodeB & 0xff;
    aLocalContactNormal = faceIndex switch
    {
        0 => new float3(1f, 0f, 0f),
        1 => new float3(0f, 1f, 0f),
        2 => new float3(0f, 0f, 1f),
        3 => new float3(-1f, 0f, 0f),
        4 => new float3(0f, -1f, 0f),
        5 => new float3(0f, 0f, -1f),
        _ => default
    };
    aLocalContactNormal = -math.rotate(bInATransform.rot, aLocalContactNormal);
    break;
}
case 8: // A face and B point
case 9: // A face and B edge
case 10: // A face and B face
{
    // For B edge and face, this can only happen due to some bizarre precision issues.
    // But we'll handle it anyways by just using the face normal of A.
    var faceIndex = distanceResult.featureCodeA & 0xff;
    aLocalContactNormal = faceIndex switch
    {
        0 => new float3(1f, 0f, 0f),
        1 => new float3(0f, 1f, 0f),
        2 => new float3(0f, 0f, 1f),
        3 => new float3(-1f, 0f, 0f),
        4 => new float3(0f, -1f, 0f),
        5 => new float3(0f, 0f, -1f),
        _ => default
    };
    break;
}
```

Next, we have two edges. In such a case, the normal is cross product between the
two edges, aligned in the direction of the first edge’s closest point normal.

```csharp
var edgeDirectionIndexA = (distanceResult.featureCodeA >> 2) & 0xff;
var edgeDirectionA = edgeDirectionIndexA switch
{
    0 => new float3(1f, 0f, 0f),
    1 => new float3(0f, 1f, 0f),
    2 => new float3(0f, 0f, 1f),
    _ => default
};
var edgeDirectionIndexB = (distanceResult.featureCodeB >> 2) & 0xff;
var edgeDirectionB = edgeDirectionIndexB switch
{
    0 => new float3(1f, 0f, 0f),
    1 => new float3(0f, 1f, 0f),
    2 => new float3(0f, 0f, 1f),
    _ => default
};
edgeDirectionB = math.rotate(bInATransform.rot, edgeDirectionB);
aLocalContactNormal = math.normalize(math.cross(edgeDirectionA, edgeDirectionB));
aLocalContactNormal = math.select(aLocalContactNormal, -aLocalContactNormal, math.dot(aLocalContactNormal, distanceResult.normalA) < 0f);
break;
```

Now we get to the hard ones. When one of the features is a vertex and the other
is an edge, we have multiple SAT axes that could have potentially detected this
case. At first, this seemed like a nightmare to solve, and I started having
doubts in going down the route of “matching Unity Physics”. But after playing
with some box-like shapes at my desk, I started to develop a theory that the
vertex + edge scenario would always result in the `FaceFace()` method failing,
causing the algorithm to fall back on a contact normal based on the closest
points. So for this case, as well as the vertex and vertex case, I just jump
straight to the second attempt logic. If my assumption is wrong, the result will
still be valid (the box colliders will be processed as if they were convex
colliders) and technically more accurate. It will just lead to slightly
different simulation results.

And in case the two closest points are perfectly touching, Unity Physics chooses
to just use the up-axis for the contact normal. This seems arbitrarily silly and
bug-prone to me, so I picked an axis based on the surface normals instead.

```csharp
case 0:  // A point and B point
case 1:  // A point and B edge
case 4:  // A edge and B point
{
    // In the case of a point and an edge, I believe only one of three things will happen:
    // 1) The SAT axis is between two edges, and the planes won't project onto each other at all.
    // 2) The SAT axis will be a face axis, in which the closest point will be clipped off.
    // 3) The SAT axis is identical to the direction between the closest points.
    //
    // The first two options in Unity Physics will trigger the FaceFace() method to fail.
    // The failure will result in Unity completely nuking its in-progress manifold and using
    // the general-purpose algorithm that relies on GJK, which uses a contact normal based on
    // the axis between the closest points. The last option is effectively the same result.
    aLocalContactNormal =
        math.normalizesafe((distanceResult.hitpointB - distanceResult.hitpointA) * math.select(1f, -1f, distanceResult.distance < 0f), float3.zero);
    if (aLocalContactNormal.Equals(float3.zero))
    {
        aLocalContactNormal = math.normalize(distanceResult.normalA - distanceResult.normalB);
    }
    aLocalContactNormal = math.InverseRotateFast(aTransform.rot, aLocalContactNormal);
    usesContactDir      = true;
    break;
}
```

The rest of the code reads very similarly to Capsule vs Box except with a few
magic control flow lines. I won’t post them here, but you can look at the full
implementation in BoxBox.cs. It admittedly doesn’t look anything like the Unity
Physics implementation, because I replaced a bunch of loops and temporary state
with SIMD logic that doesn’t need loops.

Believe it or not, all the remaining contact algorithms effectively follow the
same strategies we have identified up to this point. There are still some
gotchas we’ll explore, but there won’t be anything radically different from here
on out!

Todo: This is where I am going to cut things off for now. But don’t worry, I’m
not giving up on this. Just need to take things in stride.
