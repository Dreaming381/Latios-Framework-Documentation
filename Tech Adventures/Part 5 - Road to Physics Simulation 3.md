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

## Sphere vs Triangle

One of these says “sphere”, which means we get to do this:

```csharp
public static UnitySim.ContactsBetweenResult UnityContactsBetween(in TriangleCollider triangle,
                                                                    in RigidTransform triangleTransform,
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

These easy solutions are starting to become rare, so let’s enjoy them when we
get them.

## Capsule vs Triangle

The first difference between this and Capsule vs Box is how we compute the face
and edge planes to clip the capsule segment. There’s only three vertices, and
two possible plane directions to worry about.

```csharp
internal static void BestFacePlanesAndVertices(in TriangleCollider triangle,
                                                float3 localDirectionToAlign,
                                                out simdFloat3 edgePlaneOutwardNormals,
                                                out float4 edgePlaneDistances,
                                                out Plane plane,
                                                out simdFloat3 vertices)
{
    vertices = triangle.AsSimdFloat3();
    plane    = mathex.PlaneFrom(triangle.pointA, triangle.pointB - triangle.pointA, triangle.pointC - triangle.pointA);
    if (math.dot(plane.normal, localDirectionToAlign) < 0f)
        plane = mathex.Flip(plane);

    edgePlaneOutwardNormals = simd.cross(vertices.bcab - vertices, localDirectionToAlign);  // These normals are perpendicular to the contact normal, not the plane.
    if (math.dot(edgePlaneOutwardNormals.a, vertices.c - vertices.a) > 0f)
        edgePlaneOutwardNormals = -edgePlaneOutwardNormals;
    edgePlaneDistances          = simd.dot(edgePlaneOutwardNormals, vertices.bcab);
}
```

That’s a lot less code. And the edge planes are set up such that we don’t need
to worry about the extra instance, as it matches the first edge plane.

The other gotcha with triangles is that we run the risk of our contact normal
being near parallel to the triangle’s surface. Unity detects this by comparing
the dot product between the contact normal and the triangle’s normal against the
constant of 0.05f. This is less than 3 degrees from parallel to the triangle’s
surface.

```csharp
bool needsClosestPoint = math.abs(math.dot(boxLocalContactNormal, plane.normal)) < 0.05f;
```

Otherwise, the implementation is just a copy-and-paste. Not too bad!

On a side note, Unity Physics does a really weird thing where it does collision
detection by converting the triangle to capsule space (which requires 50% more
space conversion operations) and then flip the other way to do contact
generation, and then flip the results back again such that the triangle is B.

Why? I have no idea. It seems like it is throwing away performance here.

## Box vs Triangle

Once again, we run into a situation where Unity chooses to use a specialized SAT
algorithm for contacts and fall back on GJK under certain circumstances. And
consequently, we are going to use the same strategy we used for box vs box for
conditionally recovering the SAT contact axis.

The actual catch this time around is that not only do we have to watch out for
the contact normal being parallel to the triangle’s surface. And this is where
Unity Physics does something really weird again. It sets things up to project
the box edges against the triangle, and then if the triangle’s surface and
contact normal are problematic, it switches to going the other way around. What
is weird about this is that there’s never problems with this if the triangle
edges are always projected on the box face to start with. This saves some space
conversions as well as having to flip the contact manifold result. Again, free
performance.

With our schema, the box is always B in this scenario, so we’re already primed
to use the smarter projection direction. The only thing we need to do is guard
against projecting the box points against the triangle when the dot product of
the contact normal and the triangle plane is less than our magic threshold.

That just leaves computing the contact normal from the feature codes. We have to
swap the box A features with the triangle features. And well, an optimization
opportunity presents itself. We can skip quite a few space transforms by working
in box space rather than triangle space, as the triangle can be perfectly
transformed into an arbitrary space with equivalent representation.

Here's what some of that looks like:

```csharp
case 5:  // A edge and B edge
{
    var edgeDirectionIndexA = (distanceResult.featureCodeA >> 2) & 0xff;
    var edgeDirectionA      = edgeDirectionIndexA switch
    {
        0 => math.normalize(triangleInB.pointB - triangleInB.pointA),
        1 => math.normalize(triangleInB.pointC - triangleInB.pointB),
        2 => math.normalize(triangleInB.pointA - triangleInB.pointC),
        _ => default
    };
    var edgeDirectionIndexB = (distanceResult.featureCodeB >> 2) & 0xff;
    var edgeDirectionB      = edgeDirectionIndexB switch
    {
        0 => new float3(1f, 0f, 0f),
        1 => new float3(0f, 1f, 0f),
        2 => new float3(0f, 0f, 1f),
        _ => default
    };
    edgeDirectionA      = math.rotate(aInBTransform.rot, edgeDirectionA);
    aLocalContactNormal = math.normalize(math.cross(edgeDirectionA, edgeDirectionB));
    aLocalContactNormal = math.select(aLocalContactNormal, -aLocalContactNormal, math.dot(aLocalContactNormal, distanceResult.normalB) > 0f);
    break;
}

case 8:  // A face and B point
case 9:  // A face and B edge
{
    // For B edge, this can only happen due to some bizarre precision issues.
    // But we'll handle it anyways by just using the face normal of A.
    aLocalContactNormal = math.normalize(math.cross(triangleInB.pointB - triangleInB.pointA, triangleInB.pointC - triangleInB.pointA));
    aLocalContactNormal = math.select(aLocalContactNormal, -aLocalContactNormal, math.dot(aLocalContactNormal, distanceResult.normalA) < 0f);
    break;
}
```

## Triangle vs Triangle

There’s good news and bad news with this one.

The good news is that Unity uses GJK for this, so we don’t have to do some crazy
SAT axis recovery shenanigans. However, we’ll still use the feature codes to
identify point-face pairs as that’s a common case and more robust for figuring
out contact normals.

The bad news is that we have to worry about having to reverse roles when
projecting the edges of one triangle against the face of the other due to
contact normal alignment issues.

Fortunately, role reversal is actually fairly straightforward if we just copy
our original code and swap which triangle vertices and planes we use for each
piece. We also only need to do the edge clipping part, because if we decide to
perform a reverse, we already know that projecting onto triangle B is not worth
it. No space conversions necessary.

Oh, and what if both triangles have bad face normals relative to the contact
normal? Then we fail and fall back to a single point contact.

## Sphere vs Convex

We are two-thirds through. Time for a breather.

## Capsule vs Convex

This is our first time really dealing with clipping with convex colliders. And
immediately we encounter our first major challenge, which is the non-uniform
scale factor. This is something that Psyshock has but Unity Physics doesn’t, so
we’ll have to think up our own strategies for these special cases. First, let’s
identify what case we’re dealing with.

```csharp
float3 invScale = math.rcp(convex.scale);
var dimensions = math.countbits(math.bitmask(new bool4(math.isfinite(invScale), false)));
```

0 dimensions is easy. Our convex mesh collapsed down to a single point, which
means we only have a single contact point.

```csharp
else if (dimensions == 0) // Code is written so 3 dim is checked first as that is way more likely
{
    result.Add(distanceResult.hitpointB, distanceResult.distance);
    return result;
}
```

For one dimension, our convex hull becomes a line along an axis. We can convert
this line into a `0f` radius capsule and send it off to Capsule vs Capsule to
deal with. The nice part about this is that `DistanceBetween()` already
generated the correct feature code, so we just need to create the capsule
endpoints which we can gather from the `Aabb`.

```csharp
else if (dimensions == 1)
{
    var convexCapsule = new CapsuleCollider(blob.localAabb.min * convex.scale, blob.localAabb.max * convex.scale, 0f);
    return CapsuleCapsule.UnityContactsBetween(in convexCapsule, in convexTransform, in capsule, in capsuleTransform, in distanceResult);
}
```

For 2D, we go from having a convex structure to a point cloud soup. Originally,
I wrote an implementation that solved this particular case. But future me
learned this was going to become a persistent problem, and came up with some new
data structures in our convex hull to deal with them. We now have three 2D
convex hulls represented as a sequence of vertex indices from our 3D convex
hull. The planar convex hull builder algorithm I implemented isn’t necessarily
the fastest at building 2D hulls, but it will serve another purpose later, so I
will discuss it now.

The algorithm starts by searching for two points that are on the 2D convex hull
boundary and are far apart from each other.

```csharp
public static int Build(ref Span<byte> finalVertexIndices, ReadOnlySpan<float3> projectedVertices, float3 normal)
{
Span<UnhomedVertex> unhomed       = stackalloc UnhomedVertex[projectedVertices.Length];
int                 unhomedCount  = projectedVertices.Length;
Span<Plane>         edges         = stackalloc Plane[finalVertexIndices.Length];
int                 foundVertices = 0;

float bestDistance = 0f;
byte  bestIndex    = 0;
unhomed[0]         = new UnhomedVertex { vertexIndex = 0 };
for (byte i = 1; i < projectedVertices.Length; i++)
{
    unhomed[i] = new UnhomedVertex { vertexIndex = i };
    var newDist                                  = math.distancesq(projectedVertices[0], projectedVertices[i]);
    if (newDist > bestDistance)
    {
        bestDistance = newDist;
        bestIndex    = i;
    }
}

finalVertexIndices[0] = bestIndex;
foundVertices++;
unhomedCount--;
unhomed[bestIndex] = unhomed[unhomedCount];

bestDistance = 0f;
bestIndex    = 0;
for (byte i = 0; i < unhomedCount; i++)
{
    var newDist = math.distancesq(projectedVertices[unhomed[i].vertexIndex], projectedVertices[finalVertexIndices[0]]);
    if (newDist > bestDistance)
    {
        bestDistance = newDist;
        bestIndex    = i;
    }
}

finalVertexIndices[1] = bestIndex;
foundVertices++;
unhomedCount--;
unhomed[bestIndex] = unhomed[unhomedCount];
```

At this point, we have a segment, and that segment has two edge planes facing
opposite from each other. Every remaining planar convex hull vertex is on the
positive side of one of these two edge planes. We figure out each one and how
far away each point is from the edge.

```csharp
edges[0] = mathex.PlaneFrom(projectedVertices[finalVertexIndices[0]], projectedVertices[finalVertexIndices[1]] - projectedVertices[finalVertexIndices[0]], normal);
edges[1] = mathex.Flip(edges[0]);

for (byte i = 0; i < unhomedCount; i++)
{
    ref var u      = ref unhomed[i];
    var     vertex = projectedVertices[u.vertexIndex];
    var     dist   = mathex.SignedDistance(edges[0], vertex);
    if (dist > 0f)
    {
        u.distanceToEdge   = dist;
        u.closestEdgeIndex = 0;
    }
    else if (dist < 0f)
    {
        u.distanceToEdge   = -dist;
        u.closestEdgeIndex = 1;
    }
    else // Colinear points are removed
    {
        unhomedCount--;
        if (i < unhomedCount)
        {
            unhomed[i] = unhomed[unhomedCount];
            i--;
        }
    }
}
```

At this point, we are ready to start our loop to build out the rest of the
convex hull. We start by finding the vertex not in the hull that has the largest
distance to the hull, and we insert it into the hull.

```csharp
while (foundVertices < finalVertexIndices.Length && unhomedCount > 0)
{
    bestIndex    = 0;
    bestDistance = unhomed[0].distanceToEdge;
    for (byte i = 1; i < unhomedCount; i++)
    {
        var u = unhomed[i];
        if (u.distanceToEdge > bestDistance)
        {
            bestDistance = u.distanceToEdge;
            bestIndex    = i;
        }
    }

    var bestU       = unhomed[bestIndex];
    var targetIndex = bestU.closestEdgeIndex + 1;
    for (int i = foundVertices - 1; i >= targetIndex; i--)
    {
        finalVertexIndices[i + 1] = finalVertexIndices[i];
        edges[i + 1]              = edges[i];
    }

    finalVertexIndices[targetIndex] = bestU.vertexIndex;
    float3 before                   = projectedVertices[finalVertexIndices[bestU.closestEdgeIndex]];
    float3 current                  = projectedVertices[bestU.vertexIndex];
    float3 after                    = projectedVertices[finalVertexIndices[math.select(targetIndex + 1, 0, targetIndex >= foundVertices)]];
    edges[targetIndex]              = mathex.PlaneFrom(current, after - current, normal);
    edges[bestU.closestEdgeIndex]   = mathex.PlaneFrom(before, current - before, normal);
    foundVertices++;
    unhomedCount--;
    unhomed[bestIndex] = unhomed[unhomedCount];
```

This operation effectively divides one of the edges with two. We then loop
through all the remaining vertices and update them. Vertices that associated
with an edge greater than the one we split need to have their edge index
incremented to account for the insertion. And vertices that were associated with
the edge need to be reevaluated against the two new edges, potentially being
discarded if now inside the convex hull.

```csharp
    for (byte i = 0; i < unhomedCount; i++)
    {
        ref var u = ref unhomed[i];
        if (u.closestEdgeIndex > bestU.closestEdgeIndex)
            u.closestEdgeIndex++;
        else if (u.closestEdgeIndex == bestU.closestEdgeIndex)
        {
            var distA = mathex.SignedDistance(edges[u.closestEdgeIndex], projectedVertices[u.vertexIndex]);
            var distB = mathex.SignedDistance(edges[targetIndex], projectedVertices[u.vertexIndex]);
            if (distA <= 0f && distB <= 0f)
            {
                unhomedCount--;
                if (i < unhomedCount)
                {
                    unhomed[i] = unhomed[unhomedCount];
                    i--;
                }
            }
            else if (distB > distA)
            {
                u.distanceToEdge   = distB;
                u.closestEdgeIndex = (byte)targetIndex;
            }
            else
                u.distanceToEdge = distA;
        }
    }
}

return foundVertices;
```

And that’s the algorithm. It isn’t amazingly fast at building full convex hulls
in 2D, but it is plenty good enough for baking. However, where it shines is that
when we have a limited number of output vertices, this algorithm attempts to
optimize the area of the polygon it produces while still using the original
vertices.

Our 2D polygon hull ultimately becomes our convex face that can otherwise be
processed the same as the 3D case with slightly different vertex indexing. So
let’s explore how 3D works.

Our first step in 3D is to find the best face on the convex collider to use.
However, a face in a convex collider could have well over 4 edges. We can’t
return the edge planes in our SIMD format anymore. Instead, we’ll return the
face index and the edge count to iterate in a loop. We’ll also use feature codes
to hone in on our plane quicker. If the feature code is a face, then our result
is that face.

```csharp
internal static void BestFacePlane(ref ConvexColliderBlob blob, float3 localDirectionToAlign, ushort featureCode, out Plane facePlane, out int faceIndex, out int edgeCount)
{
    var featureType = featureCode >> 14;
    if (featureType == 2)
    {
        // Feature is face. Grab the face.
        faceIndex = featureCode & 0x3fff;
        facePlane = new Plane(new float3(blob.facePlaneX[faceIndex], blob.facePlaneY[faceIndex], blob.facePlaneZ[faceIndex]), blob.facePlaneDist[faceIndex]);
        edgeCount = blob.edgeIndicesInFacesStartsAndCounts[faceIndex].y;
    }
```

If our closest feature is an edge, we know the closest face is one of the two
faces connected to that edge. But how do we know which faces are connected to an
edge?

Prior to writing these algorithms, you would have had to search through all the
faces looking for ones which referenced the edge in question. However, I added
new index lookups into the convex collider blob for both vertices and edges to
look up attached faces. Now we just have to load up each face and determine
which is the best.

```csharp
else if (featureType == 1)
{
    // Feature is edge. One of two faces is best.
    var edgeIndex   = featureCode & 0x3fff;
    var faceIndices = blob.faceIndicesByEdge[edgeIndex];
    var facePlaneA  =
        new Plane(new float3(blob.facePlaneX[faceIndices.x], blob.facePlaneY[faceIndices.x], blob.facePlaneZ[faceIndices.x]), blob.facePlaneDist[faceIndices.x]);
    var facePlaneB =
        new Plane(new float3(blob.facePlaneX[faceIndices.y], blob.facePlaneY[faceIndices.y], blob.facePlaneZ[faceIndices.y]), blob.facePlaneDist[faceIndices.y]);
    if (math.dot(localDirectionToAlign, facePlaneA.normal) >= math.dot(localDirectionToAlign, facePlaneB.normal))
    {
        faceIndex = faceIndices.x;
        facePlane = facePlaneA;
        edgeCount = blob.edgeIndicesInFacesStartsAndCounts[faceIndex].y;
    }
    else
    {
        faceIndex = faceIndices.y;
        facePlane = facePlaneB;
        edgeCount = blob.edgeIndicesInFacesStartsAndCounts[faceIndex].y;
    }
}
```

Lastly, if the closest point is a vertex, then we have to loop through several
faces to find the best.

```csharp
else
{
    // Feature is vertex. One of adjacent faces is best.
    var vertexIndex        = featureCode & 0x3fff;
    var facesStartAndCount = blob.faceIndicesByVertexStartsAndCounts[vertexIndex];
    faceIndex              = blob.faceIndicesByVertex[facesStartAndCount.x];
    facePlane              = new Plane(new float3(blob.facePlaneX[faceIndex], blob.facePlaneY[faceIndex], blob.facePlaneZ[faceIndex]), blob.facePlaneDist[faceIndex]);
    float bestDot          = math.dot(localDirectionToAlign, facePlane.normal);
    for (int i = 1; i < facesStartAndCount.y; i++)
    {
        var otherFaceIndex = blob.faceIndicesByVertex[facesStartAndCount.x + i];
        var otherPlane     = new Plane(new float3(blob.facePlaneX[otherFaceIndex], blob.facePlaneY[otherFaceIndex], blob.facePlaneZ[otherFaceIndex]),
                                        blob.facePlaneDist[otherFaceIndex]);
        var otherDot = math.dot(localDirectionToAlign, otherPlane.normal);
        if (otherDot > bestDot)
        {
            bestDot   = otherDot;
            faceIndex = otherFaceIndex;
            facePlane = otherPlane;
        }
    }
    edgeCount = blob.edgeIndicesInFacesStartsAndCounts[faceIndex].y;
}
```

I’ll admit, the data structure here is not the friendliest in terms of random
memory accesses. But we aren’t looping through all the faces like we had to when
finding the closest points. It is better that our data structure is friendly to
those more expensive algorithms.

Convex colliders could be “sharp” like an axe head. Therefore, we need to guard
against faces nearly parallel to the contact normal like we do for triangles. We
end up with this setup piece:

```csharp
var convexLocalContactNormal = math.InverseRotateFast(convexTransform.rot, -distanceResult.normalB);
PointRayConvex.BestFacePlane(ref convex.convexColliderBlob.Value, convexLocalContactNormal * invScale, distanceResult.featureCodeA, out var plane, out int faceIndex, out int edgeCount);

bool needsClosestPoint = math.abs(math.dot(convexLocalContactNormal, plane.normal)) < 0.05f;

if (!needsClosestPoint)
{
    var bInATransform = math.mul(math.inverse(convexTransform), capsuleTransform);
    var rayStart = math.transform(bInATransform, capsule.pointA);
    var rayDisplacement = math.transform(bInATransform, capsule.pointB) - rayStart;
```

The very last step is to perform our edge plane raycasts so that we can get the
two fractions to clip our capsule’s segment. If we want to preserve our SIMD
raycasting, we’ll need to process four edges at a time, potentially with
wraparound (processing an edge twice is harmless). Another concern is ensuring
the edge planes all face outward. This is further complicated by edge directions
in a face being arbitrarily ordered. Fortunately, because a face is a convex
polygon, the average point of all the midpoints of any set of edges (including
repeats) should always lie in the interior of the face. We can then use a dot
product test to correct the signs.

```csharp
var    edgeIndicesBase = blob.edgeIndicesInFacesStartsAndCounts[faceIndex].x;
float4 enterFractions  = float4.zero;
float4 exitFractions   = 1f;
bool4  projectsOnFace  = true;
for (int i = 0; i < edgeCount; i += 4)
{
    int4 indices       = i + new int4(0, 1, 2, 3);
    indices            = math.select(indices, indices - edgeCount, indices >= edgeCount);
    indices           += edgeIndicesBase;
    var segmentA       = blob.vertexIndicesInEdges[blob.edgeIndicesInFaces[indices.x].index];
    var segmentB       = blob.vertexIndicesInEdges[blob.edgeIndicesInFaces[indices.y].index];
    var segmentC       = blob.vertexIndicesInEdges[blob.edgeIndicesInFaces[indices.z].index];
    var segmentD       = blob.vertexIndicesInEdges[blob.edgeIndicesInFaces[indices.w].index];
    var edgeVerticesA  = new simdFloat3(new float4(blob.verticesX[segmentA.x], blob.verticesX[segmentB.x], blob.verticesX[segmentC.x],
                                                    blob.verticesX[segmentD.x]),
                                        new float4(blob.verticesY[segmentA.x], blob.verticesY[segmentB.x], blob.verticesY[segmentC.x],
                                                    blob.verticesY[segmentD.x]),
                                        new float4(blob.verticesZ[segmentA.x], blob.verticesZ[segmentB.x], blob.verticesZ[segmentC.x],
                                                    blob.verticesZ[segmentD.x]));
    var edgeVerticesB = new simdFloat3(new float4(blob.verticesX[segmentA.y], blob.verticesX[segmentB.y], blob.verticesX[segmentC.y],
                                                    blob.verticesX[segmentD.y]),
                                        new float4(blob.verticesY[segmentA.y], blob.verticesY[segmentB.y], blob.verticesY[segmentC.y],
                                                    blob.verticesY[segmentD.y]),
                                        new float4(blob.verticesZ[segmentA.y], blob.verticesZ[segmentB.y], blob.verticesZ[segmentC.y],
                                                    blob.verticesZ[segmentD.y]));
    edgeVerticesA *= convex.scale;
    edgeVerticesB *= convex.scale;

    // The average of all 8 vertices is the average of all the edge midpoints, which should be a point inside the face
    // to help get the correct outward edge planes.
    var midpoint           = simd.csumabcd(edgeVerticesA + edgeVerticesB) / 8f;
    var edgeDisplacements  = edgeVerticesB - edgeVerticesA;
    var edgePlaneNormals   = simd.cross(edgeDisplacements, convexLocalContactNormal);
    edgePlaneNormals       = simd.select(-edgePlaneNormals, edgePlaneNormals, simd.dot(edgePlaneNormals, midpoint - edgeVerticesA) > 0f);
    var edgePlaneDistances = simd.dot(edgePlaneNormals, edgeVerticesB);

    var rayRelativeStarts  = simd.dot(rayStart, edgePlaneNormals) - edgePlaneDistances;
    var relativeDiffs      = simd.dot(rayDisplacement, edgePlaneNormals);
    var rayRelativeEnds    = rayRelativeStarts + relativeDiffs;
    var rayFractions       = math.select(-rayRelativeStarts / relativeDiffs, float4.zero, relativeDiffs == float4.zero);
    var startsInside       = rayRelativeStarts <= 0f;
    var endsInside         = rayRelativeEnds <= 0f;
    projectsOnFace        &= startsInside | endsInside;
    enterFractions         = math.select(float4.zero, rayFractions, !startsInside & rayFractions > enterFractions);
    exitFractions          = math.select(1f, rayFractions, !endsInside & rayFractions < exitFractions);
}

var fractionA     = math.cmax(enterFractions);
var fractionB     = math.cmin(exitFractions);
needsClosestPoint = true;
if (math.all(projectsOnFace) && fractionA < fractionB)
{
    // Add the two contacts from the possibly clipped segment
    var distanceScalarAlongContactNormal = math.rcp(math.dot(convexLocalContactNormal, plane.normal));
    var clippedSegmentA                  = rayStart + fractionA * rayDisplacement;
    var clippedSegmentB                  = rayStart + fractionB * rayDisplacement;
    var aDistance                        = mathex.SignedDistance(plane, clippedSegmentA) * distanceScalarAlongContactNormal;
    var bDistance                        = mathex.SignedDistance(plane, clippedSegmentB) * distanceScalarAlongContactNormal;
    result.Add(math.transform(convexTransform, clippedSegmentA), aDistance - capsule.radius);
    result.Add(math.transform(convexTransform, clippedSegmentB), bDistance - capsule.radius);
    needsClosestPoint = math.min(aDistance, bDistance) > distanceResult.distance + 1e-4f;  // Magic constant comes from Unity Physics
}
```

Is it a lot of code?

Yes.

Can it be improved with a smarter convex hull representation?

Probably.

There’s still 3 more pairs to go. They don’t get any easier.

## Box vs Convex

We’ve encountered so many scenarios at this point. Which techniques can we apply
here?

We have a convex collider. This means a few things. On one hand, we have to
worry about dimensionality. On the other, Unity never uses SAT for convex
colliders, so we can use our simpler logic for finding the contact normal. We
also have a box collider, which means we don’t have to worry about failure to
project convex edges against the box. But unlike Triangle vs Box, convex
colliders can have potentially many more vertices than the box, so we’ll be
working in the convex collider space. This also proposes another problem, which
is that we could have up to 37 contact points in 3D and even more in 2D, but our
output struct can only hold 32. We can create a wrapper struct that adds
additional storage to solve this, and then use our polygon expansion algorithm
to bring our count back down.

It is actually 1-dimensional convex colliders that give us our first surprise.
In 1D, our convex collider turns into a capsule. This means we have a capsule vs
box, but our method for handling this requires the arguments to be flipped. If
we want to leverage such a method, we need to flip our `ColliderDistanceResult`
and then flip the resulting `ContactsBetweenResult`. The latter can be done with
this little method:

```csharp
public void FlipInPlace()
{
    for (int i = 0; i < contactCount; i++)
    {
        var contact       = this[i];
        contact.location += contact.distanceToA * contactNormal;
        this[i]           = contact;
    }
    contactNormal = -contactNormal;
} 
```

In 2D, we need to project our convex edges against the box face, which is easy
to do as we can just process each edge one at a time and use SIMD on the box.
Our previous algorithms already worked this. Where things get tricky however is
projected the box vertices onto the convex mesh face. In the past, we tested
each vertex one by one against all the edge planes. That was fine when we could
cover all the edge planes at once. But convex collider faces can have more than
four edges. And those edges aren’t the most trivial to assemble from the blob. A
better approach would be to test all four box vertices against each edge inside
the same loop where we test the edge against the box plane. For this, we need to
keep track of whether each box vertex is on the “inside” side of the edge plane.
But is that the positive side or the negative side?

It actually doesn’t matter. If a point is always on the same side, and are edge
planes are consistent, then we know that the point must be inside. And to
simplify the code a little bit, we can use the fact that if the point is always
on one side when it is inside, then it is never on the other side when it is
inside. If we tally both sides, and determine that one side never receives a
hit, then we know it is inside. That code looks like this:

```csharp
    // Inside for each edge of A
    if (projectBOnA)
    {
        var aEdgePlaneNormal   = math.cross(rayDisplacement, aLocalContactNormal);
        var edgePlaneDistance  = math.dot(aEdgePlaneNormal, rayStart);
        var projection         = simd.dot(bVertices, aEdgePlaneNormal) + edgePlaneDistance;
        positiveSideCounts    += math.select(int4.zero, 1, projection > 0f);
        negativeSideCounts    += math.select(int4.zero, 1, projection < 0f);
    }
}
if (projectBOnA)
{
    var distanceScalarAlongContactNormalA = math.rcp(math.dot(aLocalContactNormal, aPlane.normal));
    var bProjectsOnA                      = math.min(positiveSideCounts, negativeSideCounts) == 0;
    for (int i = 0; i < 4; i++)
    {
        var vertex = bVertices[i];
        if (bProjectsOnA[i])
        {
            var distance = mathex.SignedDistance(aPlane, vertex) * distanceScalarAlongContactNormalA;
            result.Add(math.transform(convexTransform, vertex), distance);
            needsClosestPoint &= distance > distanceResult.distance + 1e-4f;
        }
    }
}
```

The last step is to handle the possible overflow of contact points. Our wrapper
struct looks like this:

```csharp
unsafe struct UnityContactManifoldExtra2D
{
    public UnitySim.ContactsBetweenResult baseStorage;
    public fixed float                    extraContactsData[384];

    public ref UnitySim.ContactsBetweenResult.ContactOnB this[int index]
    {
        get
        {
            if (index < 32)
            {
                fixed (void* ptr = baseStorage.contactsData)
                return ref ((UnitySim.ContactsBetweenResult.ContactOnB*)ptr)[index];
            }
            else
            {
                fixed (void* ptr = extraContactsData)
                return ref ((UnitySim.ContactsBetweenResult.ContactOnB*)ptr)[index - 32];
            }
        }
    }

    public void Add(float3 locationOnB, float distanceToA)
    {
        this[baseStorage.contactCount] = new UnitySim.ContactsBetweenResult.ContactOnB { location = locationOnB, distanceToA = distanceToA };
        baseStorage.contactCount++;
    }
}
```

If we don’t spill contacts into the extra storage, we can just return the
interior result. Otherwise, we’ll use our ExpandingPolygonBuilder2D to help us
choose which contacts to keep. Thus, we finish up 2D with this:

```csharp
var requiredContacts = math.select(32, 31, needsClosestPoint);
if (result.baseStorage.contactCount <= requiredContacts)
{
    if (needsClosestPoint)
        result.baseStorage.Add(distanceResult.hitpointB, distanceResult.distance);
    return result.baseStorage;
}

// Simplification required
Span<byte> indices = stackalloc byte[requiredContacts];
Span<float3> projectedContacts = stackalloc float3[result.baseStorage.contactCount];
for (int i = 0; i < result.baseStorage.contactCount; i++)
{
    projectedContacts[i] = result[i].location;
}
var finalVertexCount = ExpandingPolygonBuilder2D.Build(ref indices, projectedContacts, bPlane.normal);
UnitySim.ContactsBetweenResult finalResult = default;
finalResult.contactNormal = result.baseStorage.contactNormal;
for (int i = 0; i < finalVertexCount; i++)
{
    finalResult.Add(result[indices[i]]);
}
if (needsClosestPoint)
    finalResult.Add(distanceResult.hitpointB, distanceResult.distance);
return finalResult;
```

As for 3D, well that is exactly the same as 2D, except that it has to pull the
edge vertices a little differently. The implementation feels surprisingly clean
and fast.

## Triangle vs Convex

Triangle vs Convex has six scenarios, four of which are identical to Box vs
Convex. The other two scenarios come from when the triangle needs to project
edges against the convex face rather than the other way around. This applies to
both 2D and 3D scenarios. We’ll start from the Box vs Convex code, insert the
checks for which direction we should do edge projection, and then work out the
cases we haven’t handled out.

When we project triangle edges onto the convex face, we don’t have to worry
about projecting convex vertices back onto the triangle. At this point, the
situation starts to resemble Capsule vs Convex, except we have three edges
instead of one, and we have to account for `fractionB` to avoid duplicates. So
why not just do that? Inside our loop through all the convex edges
four-at-a-time, instead of testing a single capsule edge, we test all three
triangle edges? We keep three copies of the enter fractions, the exit fractions,
and the validity statuses. It becomes a lot of copy-and-paste with a few
swapped-out variable names, but solves the problem without having to invent
anything new.

One more pair to go.

## Convex vs Convex

Convex vs Convex is very similar to Triangle vs Convex, except now the triangle
is also a convex collider. And now instead of 4 dimensions, we have 4x4
dimensions. This one definitely feels like a boss fight.

Todo: This is where I am going to cut things off for now. But don’t worry, I’m
not giving up on this. Just need to take things in stride.
