# Tech Adventures: Road to Physics Simulation Part 4

There are two kinds of game engine programmers. There’s those who understand
math notation in papers, and those who don’t.

I fall into the latter camp. While I know a few things, most of the time I
struggle to keep track of which arbitrarily assigned letter maps to which
property being discussed. And then all these formulas, as well as the sequences
the formulas are applied, are all given names after various mathematicians and
researchers in combination of abstract terminology such as “implicit” or
“projected” that are complete gibberish next to people names until you dig
several layers deep.

When I do 3D mathematics and physics things, I visualize everything in a 3D
space. The reason I make so few visuals is because I don’t need to, they already
exist in my head when I reason about the problem. If I had a way to project my
thoughts into digital images, these pages would be full of visual diagrams. But
until there’s a tool that makes it relatively quick and easy to create 3D
diagrams with markup lines and labels, I stick mostly to words.

My way of going about things is not popular in the math and physics worlds,
which is why I have no idea which algorithms Unity is using for their
simulation, nor can I make sense of it from the code. From reading the code, it
seems they use some kind of Jacobian solver, given how often the term shows up
in the code. But then they also have “Trigger Jacobians” which is completely
nonsensical, so the use of the term could be legacy BS. I do know they also use
graph coloring for their “phase dispatcher” which from my reading shouldn’t be
necessary for a Jacobian solver.

The good news is that we may not need to figure this out in order to reproduce
the solver. After all, our goal is to fix the higher-level architecture in
Psyshock. Psyshock already uses several of Unity Physics other algorithms such
as the convex hull builder, GJK, and EPA. And while I have a better
understanding of these algorithms, I couldn’t explain them line-by-line. In this
adventure, we’re going to port these algorithms into a new architecture with
only a rough understanding of what they do, and maybe we’ll discover some clues
along the way. Perhaps in a future update, we’ll make more sense of them, or
maybe one of you readers out there might be able to enlighten me.

## Decoupling Strategies

One of the challenges with Unity Physics is that the core algorithms are baked
into the architecture, and the various pieces of code are aware of high-level
features, hence it has things like “Trigger Jacobians”. Our goal is to cut out
all that feature fluff and isolate the critical pieces of data and logic into
small building blocks that anyone could use to build their own physics engine.
But how do we determine what is critical?

One thing working in our favor is that Unity Physics is stateless. That means
that all the data goes in, there’s a sequence of transformations with temporary
state, and then there’s a bunch of outputs. It is effectively a directed-acyclic
graph. And like any other DAG, if we know some inputs and outputs are unused, we
can cull a bunch of nodes from the graph. We’ll identify the essential inputs
and outputs we care about, and trace them through the system to find the
essential influences along the way.

We know we don’t care about layer masks and keys and filters and all that. And
we also don’t care about broadcasting collision and trigger events. Those are
all higher-level problems that Psyshock already provides the tools to control in
a decoupled fashion. We’ll be able to ignore those. We’ll also try to remove any
data indexing pieces. Some of the types and values may be specific to how Unity
Physics tracks data through its various `NativeStreams`. We won’t need those
either.

The other thing working to our benefit is that Unity Physics has evolved their
pipeline to be more customizable. It no longer is focused on callbacks, and
instead has system groups that users can inject their own jobs into and read the
various NativeStreams. Users still have to filter through the streams to find
the data they care about, which is a performance sink that Psyshock will bypass.
But at the very least, the various checkpoints in the simulation are
well-documented, giving us a high-level roadmap to reference as we perform our
data transformation tracing. Two critical documents are as follows:

<https://docs.unity3d.com/Packages/com.unity.physics@1.2/manual/concepts-simulation.html>

<https://docs.unity3d.com/Packages/com.unity.physics@1.2/manual/physics-pipeline.html>

There’s a few things we can take away from this.

First, we know that Unity applies integration as the last step of its pipeline.
Some physics engines do this first, while others do this last. Knowing which we
want to replicate is helpful.

Second, we’ve already covered some of the phases. `PhysicsCreateBodyPairsGroup`
is the job of FindPairs. And `PhysicsCreateContactsGroup` is a combination of
`Physics.DistanceBetween()` and `UnitySim.ContactsBetween()`. You might already
have spotted a flaw in Unity’s design here, as simply knowing whether two
objects overlap has to go through contact generation. Maybe there’s an
early-out? Even if there is, it still smells.

The next stage however is the `PhysicsCreateJacobiansGroup`. We don’t have
anything like that yet, so that’s our next target. We’ll look at what data that
needs, what data that spits out, and try to figure out how much of that data we
care about.

## To Split or Not to Split?

Our journey begins in Unity’s `IJacobiansJob`, where we try to identify what
kinds of data Unity exports as outputs. We can see that as mentioned before,
there are these “Jacobian” types that are stored all intermixed inside a single
`NativeStream`. Consequently, there are header types that help identify and
dispatch the correct type to the right methods. Curiously, the job type only
provides to the users contact and trigger types, even though there seem to be
more in the stream. Let’s investigate this further by jumping into Jacobians.cs.

Right away, we are greeted by an `enum` specifying all the various types. These
are broken up into three categories: contacts, joints, and motors. Then we have
another `enum` for flags, where some of these flags apply to all types, and some
only to certain types. And finally, we get the `JacobianHeader`, which contains
the pair of bodies involved as well as the types and flags.

Here’s our first challenge, do we even need `JacobianHeader` in Psyshock? Or can
we just keep all the types separate?

Well, there are only two flags that apply to all types, and those are the
`Disabled` flag, and the `EnableMassFactors` flag. Psyshock’s architecture has
the policy of “don’t execute unless called”, so a `Disabled` flag isn’t very
useful. That just leaves `EnableMassFactors`, which can be traced all the way
back to Material.cs where the XML documents it as follows:

>   If true, the object can have its inertia and mass overridden during solving.

Basically, we just have to ensure our API allows setting the mass values, and we
don’t have to worry about this. And that means we can focus exclusively on the
individual types. We’ll start with the most essential, contacts.

### Contacts, Masses, and Sliding

Heading over to ContactJacobian.cs, we find a `ContactJacobian` struct with some
various properties. However, before we try to make sense of them, we should take
a look at the `Solve()` method. After all, that seems like the kind of method
that would contain an algorithm we would want to extract. Just to verify, let’s
see who calls it?

It gets forwarded by the `JacobianHeader`, which is called by `Solver.Solve()`
which is iterating through the `NativeStream`, and that gets called by
`ParallelSolverJob`, which gets scheduled multiple times in succession depending
on the number of solver iterations required. So yeah. It is safe to say this is
an essential algorithm that seems to handle solving a single pair of contacting
bodies once in an iteration, and affecting some other state.

However, `Solve()` also requires the `JacobianHeader`. Turns out,
`ContactJacobian` doesn’t actually contain all the data. There’s some
variable-sized and optional data, such as state for each contact point, override
masses, and surface velocities (think boosters). How do we deal with this?

Remember, our goal is to extract algorithms, and put them into static methods.
We can simply ask for each piece as a separate argument in our static method.
And our static method is going to do what `Solve()` does, or more specifically,
what `SolveContact()` does, since we don’t care about triggers (two kinematic
bodies).

Actually, what is `SolveInfMassPair()` doing? What is there to “solve”?

## Unity Physics Dirty Secret

Here’s the method in question:

```csharp
// Solve the infinite mass pair Jacobian
public void SolveInfMassPair(
    ref JacobianHeader jacHeader, MotionVelocity velocityA, MotionVelocity velocityB,
    Solver.StepInput stepInput, ref NativeStream.Writer collisionEventsWriter)
{
    // Infinite mass pairs are only interested in collision events,
    // so only last iteration is performed in that case
    if (!stepInput.IsLastIteration)
    {
        return;
    }

    // Calculate normal impulses and fire collision event
    // if at least one contact point would have an impulse applied
    for (int j = 0; j < BaseJacobian.NumContacts; j++)
    {
        ref ContactJacAngAndVelToReachCp jacAngular = ref jacHeader.AccessAngularJacobian(j);

        float relativeVelocity = BaseContactJacobian.GetJacVelocity(BaseJacobian.Normal, jacAngular.Jac,
            velocityA.LinearVelocity, velocityA.AngularVelocity, velocityB.LinearVelocity, velocityB.AngularVelocity);
        float dv = jacAngular.VelToReachCp - relativeVelocity;
        if (jacAngular.VelToReachCp > 0 || dv > 0)
        {
            // Export collision event only if impulse would be applied, or objects are penetrating
            ExportCollisionEvent(0.0f, ref jacHeader, ref collisionEventsWriter);

            return;
        }
    }
}
```

The variable `VelToReachCp` is an abbreviation of “velocity to reach the contact
plane”. What that means, is that the way Unity Physics is detecting collision
events is by predicting if the contact point pairs will become penetrating
during integration, or would experience a bounce impulse if they weren’t both
kinematic. This means Unity Physics relies on contact points just to figure out
if two kinematic bodies are overlapping, and is therefore calculating contact
points for all kinematic bodies.

But if you scroll down further in the file, you’ll find triggers use the exact
same algorithm!

Why?

That’s so much unnecessary calculation, especially if all you really care about
are discrete overlaps, which is really common for game logic. This is one of the
big reasons people complain Unity Physics performs so poorly. Most people use
collider shapes for simple spatial queries, sometimes just for checking with
raycasts and such. And yet all these shapes are generating contacts with each
other just to determine whether or not there are collision or trigger events,
when most of the time, another simpler broadphase pass and some cheaper overlap
tests would suffice.

Up to this point, I always assumed the excessive generation of contacts was
because they were being lazy with their code. But now I know that’s not the
case. Instead, there are trying to encapsulate a complexity of simple concerns,
only to end up fighting themselves. This isn’t a problem that can be fixed
easily. I wish the Unity Physics developers the best in trying to solve this
problem. They designed themselves into a corner here, and I would not be
surprised if the person who made that design choice is no longer around.

Fortunately, Psyshock’s architecture is completely immune to this kind of
problem. FindPairs allows for a granular separation of concerns between shapes,
so triggers and simulation can be handled independently and don’t need to affect
each other’s performance. With that said, I suspect we will still need projected
collision events for non-kinematic bodies, so we should make sure our algorithm
is capable of producing that information.

## Porting a Solve

We are going to start by porting this `SolveContact()` method, and then expand
outward from there. And so, to start with, we are going to copy and paste the
body of the method into a new static method without any arguments, and work
through step-by-step to eliminate the errors.

### The First Loop – Solve Normal Impulses

The first piece is the `MotionVelocity`. In Unity Physics, this contains a
combination of static and dynamic data such as velocity, mass, inertial, gravity
multipliers, ect. Our method only needs some of this data, so we will split
these up as separate arguments based on which are read-only and which are
read-write. By doing this, we can also expect the user to pass us in the correct
overridden mass and inertia values, so the logic to patch that data can be
dropped.

The second piece is `ContactJacAndAndVelToReachCp`. That’s quite the name, but
really, this is just some accumulating `ContactJacobian` state per contact. Even
Unity Physics has a TODO asking for a better name. We can just call it
`ContactJacobianContactState`. We’ll copy this and its embedded type
`ContactJacobianAngular` over as-is. We’ll then need a span of these as an
argument, one for each contact point.

Next, there’s a `GetJacVelocity()` static method we can just copy over
internally. However, one of its arguments is a `BaseContactJacobian.Normal`.
This is actual the normal from `UnityContacts`. We’ll add that normal to the
list of arguments, and maybe find a better home for it later.

Then we have `ApplyImpulse()`, which needs a `MotionStabilizationInput` per
body. I’ll need a new name for this type as well, but the mass renaming will
come later once we figure out what other motion stabilization types we need and
how we want to disambiguate them. As for the method, it actually makes several
calls to `MotionVelocity` for applying the impulses modifying velocity while
reading inverse mass and inertia. However, those methods are singular
multiplications, so we’ll just inline them manually. That does mean we’ll need
those as arguments.

### Contact Events

The next piece is handling contact events. Unity Physics writes this out to a
NativeStream, copying all the contact points (which is the only time this method
reads contact points). The only actual data computed directly in this method is
whether the event should be fired and the accumulated impulses. The former can
be this method’s `return` value, and the latter can be an `out` parameter.

### Solve Friction

Todo: This is where I am going to cut things off for now. But don’t worry, I’m
not giving up on this. Just need to take things in stride.
