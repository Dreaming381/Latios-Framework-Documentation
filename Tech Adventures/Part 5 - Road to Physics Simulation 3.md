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
specific. But for now, it is time to enter the 15 scenario gauntlet once again.

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

Todo: This is where I am going to cut things off for now. But don’t worry, I’m
not giving up on this. Just need to take things in stride.
