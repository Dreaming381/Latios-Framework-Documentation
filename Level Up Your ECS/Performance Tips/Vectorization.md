# Level Up Your ECS - Vectorization

Hi everyone!

Welcome to the **performance** series of Level Up Your ECS, where I deep-dive
topics of performance in an ECS context, providing various tips, and clearing up
various misconceptions. If you are looking for more example-driven discussions
of performance, check out the Optimization Adventures series.

This time, we’ll be discussing vectorization, a topic that is often
misunderstood by many developers.

## What is Vectorization?

If you’ve never heard of vectorization, or SIMD, or any of those terms, it
refers to the ability of full-fledged microprocessors to compute multiple values
within a single instruction. This means more computation in the same amount of
time, which means faster completion of calculations.

ARM-based processors used in mobile devices can compute 4 floating point
products at once in nearly the same amount of time to compute just 1. And modern
x86-based PCs can do even better with 8 or even 16 products at once.

If this sounds familiar to you, good. If not, take some time to read up on other
resources to at least get the very basic foundation of what these instructions
are at a conceptual level.

## You’ve Probably Been Misinformed

Have you ever seen someone claim that Burst automatically vectorizes the code to
make it 4 or 8 times faster? Have you ever seen someone claim that because they
use the Burst compiler, their code is vectorized, or uses SIMD, ect?

Such individuals probably have no idea what they are talking about. This used to
be prominently advertised in various assets, marketing materials, and tech
gossip circles across the internet. Fortunately, that language has declined
recently, but I still encounter it once in a while.

The reality is that in common practice, Burst does not automatically vectorize
your code. The initial marketing when Burst was released wasn’t that Burst
automatically made your code faster by vectorizing it, it was that Burst was a
compiler that was actually capable of doing this thing at all. But unless you
went out of your way to write code that Burst understands how to vectorize,
there’s a good chance none of your Burst-compiled code has been automatically
vectorized to make full use of SIMD. I repeat, it is very likely that none of
your code is vectorized.

This is both good news and bad news. The bad news is that means you aren’t
getting the best possible out of Burst. The good news is that if you are having
performance problems, there’s potentially more performance you can squeeze out.
But this certainly makes one wonder, if Burst isn’t vectorizing the code, then
where does its massive speedups come from?

Imagine you had a hot dish meal in front of you, that you were ready to eat,
except your brain is a CPU that has to run a program to instruct you to eat the
food. You have your dish, and you have some utensils of your culture
(silverware, chopsticks, ect). The program has you pick up the utensils, stir
the dish, and then set the utensils back down. Then it has you pick up the
utensils, pick out some of the dish, set it on the plate, and then put the
utensils back down. Then it has you pick up the utensils and the part of the
dish you picked out, put the food in your mouth, and then set your utensils back
down. Then after chewing and swallowing, it has you repeat this process over and
over and over again.

You’d probably look ridiculous if you actually did that.

But this is typically what the C\# compiler does. After every operation, it
always tries to put everything in a cleanly understood state so that whatever
comes next (it doesn’t know) will be able to work with what is there.

What makes Burst special is that it has a much broader view of your code. It has
foresight, hindsight, and deeper insight into all the various operations. With
this knowledge, it can cut out a lot of the fluff, and shorten the sequence of
actions taken to the minimum required. And its strict rules about only
supporting unmanaged types makes it very effective at this…

Unreasonably effective.

When Unity first released Burst, they thought it was just going to be used for
only a few key performance-sensitive jobs that needed absolute optimization.
These jobs would be hand-written by advanced developers keeping a close eye on
the Burst inspector to ensure the maximum performance possible. What instead
happened is that developers realized Burst could speed up just about anything it
was compatible with: math, decision trees, loops, method calls, properties,
anything unmanaged. Burst made it all faster with a single attribute. And
therefore, some devs started running the majority of their codebase through
Burst. The Burst team had to make quite a few optimizations to the compiler to
keep up with this much larger workload. Even the latest CoreCLR is beat by
Burst. Burst has turned out (partly by accident) to be that good.

However, for as good as it is, it is still quite far from understanding our true
intents as programmers. It doesn’t understand assumptions about memory buffers
and variables and their semantic meanings the way we do. Sometimes, it can shoot
itself in the foot assuming a loop iterates for thousands of iterations when in
practice the loop only iterates 3-5 times. Most of the time, Burst plays it
conservatively. And this is the reason that it fails to vectorize code by
default. But when things are just right, I’ve seen it completely re-architect my
algorithms, changing how the loops work and modifying the branch conditions, all
to produce an end result that is logically equivalent, but much faster.

When some people talk about Burst, they talk about the specific optimizations it
makes with the Mathematics package, and how that allows for vectorization. This
is more accurate than the previous claims, but still not the full story.

Vector operations involving the various mathematic types such as `float3` and
`quaternion` can be expressed using a combination of SIMD and scalar (normal)
instructions. And technically, using SIMD instructions for these operations is
faster. But there’s some work that needs to be done juggling data in order for
the operations to utilize the SIMD instructions, and that juggling also has a
performance cost. In the end, vector math with SIMD instructions will at most
net you a 2x speedup over doing everything scalar. It is far from optimal, but
definitely a nice bonus that is super easy to use.

It is possible to express Mathematics types such as `float4` and `int4` to
express much more optimal vectorization and break past the 2x barrier. However,
even this approach isn’t always optimal. We’ll discuss this more later.

## What Optimal Vectorization Actually Looks Like

To understand what optimal vectorization looks like, you first need to
understand the concepts of registers and instructions. A register is special CPU
memory where it is allowed to directly process with math operations. And an
instruction is the single thing it is going to do, along with which registers it
is going to do that thing on. There may be an instruction to load data from
memory into a register. An instruction to add two registers together, where the
result gets stored in one of the two registers (or possibly a third register),
and an instruction to store the register result back into memory. This is
more-or-less how ARM processors work. x86 assembly is a bit weirder for…
reasons. But what actually happens in hardware is similar enough.

For vectorization, CPUs have vector registers. These are registers that can be
loaded with adjacent memory addresses in a single instruction. And any math
instruction that operates on two normal registers typically has an associated
instruction that operates on two vector registers. There are also instructions
that operate on single registers. Some of these are math operations, which
operate on pairs of adjacent values within a single register. And some of these
instructions simply reorder the elements within the register. Unless you are
intentionally trying to reorder data in memory (rare), these reordering
instructions aren’t doing any meaningful work. They are just adapting data to
the CPU, at the cost of overhead.

If you think about it enough, math instructions that operate with two registers
are always going to get more math operations done. So a dot product between two
vectors will have a very fast initial multiplication, as the two vectors each
reside in a register and are component-by-component multiplied in a single
instruction. However, the products now live in a single register, which makes it
awkward to sum the products together. Either each of the elements need to get
pulled out into separate registers, or there needs to be a pairwise add, a
shuffling operation to get the first pair of sums next to each other, and then
another pairwise add. This is where vector math loses a lot of efficiency.

The best way to make full use of the vectorization hardware is to ensure that we
are always doing operations involving two registers. For that to work, we need
adjacent elements in a vector register to be completely unrelated to each other.
They need to be independent pieces of data that undergo the same set of
calculations. Imagine that instead of two vectors we needed to compute a dot
product for, we instead had two lists of vectors that we needed to compute a
bunch of dot products for. Except, instead of storing the ordinates adjacent in
memory, we stored each ordinate of each list in a completely separate array. We
can now run the scalar algorithm for computing a single dot product in this
setup, except by using vector registers and instructions, we can compute many
dot products for the same cost. There’s no need to shuffle data around.
Registers can simply be loaded straight from the arrays, and every computation
sets up the register for the next computation. This is what optimal
vectorization looks like.

Therefore, the key to vectorization is that it requires an SOA layout of the
primitive data types, and for computations performed on each corresponding
element of the array to be independent of the others (at least within phases).
Anything that isn’t that requires data juggling, which will eat performance and
sometimes make vectorization worse than scalar code. Vectorization is hard to do
right.

## Vectorization and ECS – They Are Not Friends

Based on that previous paragraph, you might be wondering what ECS does about
this to help you. The answer is nothing. Unity’s ECS implementation does not
encourage vectorization. In fact, it discourages it. Let’s break it down:

Imagine you have a component that represents a 3D position. Most likely, you
would express this as follows:

```csharp
struct Position3D : IComponentData
{
    public float3 position;
}
```

The main problem with this is that it is not SOA layout. Within an ECS chunk,
this is an array of structs, where each struct is three independent `float`
values. To truly make this SOA, you’d need to define three separate components
for each ordinate: x, y, and z. Yet, none of Unity’s packages do anything like
this. That’s because Unity isn’t optimizing their code for vectorization.
Instead, Unity is optimizing for cache.

Imagine you needed to do a lookup of this position from another entity. With a
`float3`, you are only performing a random access on a single cache line, and
you get all three values. Whereas if you had three separate components, you
would need to perform three separate lookups, accessing three separate cache
lines. As undesirable as random accesses are, there are many situations where
they are unavoidable. So making them less painful makes sense.

But for further justification, ECS chunk iteration rarely has the potential to
benefit from vectorization. Usually, heavy calculations involve relationships.
Code that is iterating chunks and not doing random accesses is usually
lightweight work. That is, there are very few instructions needed for the amount
of memory the chunk iteration touches. And even with scalar code, it is still
possible to reach the memory bandwidth limit with such code. It is much easier
to let users write branchy code with vector math and let Burst do the best it
can with it.

Even if you did want to define components to be vectorization-friendly, you’d
still encounter a problem, which is the small number of entities that can fit in
a chunk. Even the absolute maximum of 128 entities is 8x smaller than what
streaming hardware wants to work with when performing vector operations on
`float`s. For smaller data types, the problem only gets worse. And then you must
factor in that chunks might not be entirely full, so you need to handle awkward
sizes. And don’t even get me started on `IEnableableComponent`.

## When to Actually Use Vectorization

The reality is that most gameplay code can’t actually benefit from
vectorization. While vector math may not be the most efficient way to utilize
SIMD hardware, memory accesses and other gameplay constraints might still make
it the optimal choice. That brings the question, when should you actually
consider attempting full vectorization? There are three signs to look for. If
you see any of them in an area where you are having performance issues,
vectorization may be worth a try.

The first sign is when you have higher than O(n) algorithmic complexity. In
these cases, the cost of refitting the data into a vectorization-friendly form
can be negligible. Some example use cases include physics and pathfinding.
However, physics can be especially awkward, as often times strategically placed
branches can skip a lot of work. But such branches can disrupt vectorization
flow. Most physics engines choose to use vector math except for some very
strategic locations. Bepu Physics is one of the few physics engines that commits
fully into vectorization. The easiest use case for algorithmic complexity to
vectorize is if you have some game mechanic where every entity influences every
other entity, creating an O(n\^2) operation you can’t simplify.

The second sign is when you have a lot of to calculate for each object. This
might involve working with mesh or texture modification, handling audio,
processing animation over a skeleton, working with strings, culling against
frustum planes, dealing with dynamic buffers in general, or using some kind of
subdivision approximation for a numeric problem (such as computing the length of
a cubic Bezier curve). Some of these can be a little tricky to make the data fit
the SOA model vectorization requires. But many of these use cases often involve
enough math that converting to a vectorized form, doing the work, and then
converting back is worthwhile.

The last sign is when you happen to have a lot of scalar math and data that is
already fit in SOA layout. It is pretty rare that this happens for entities, but
can sometimes happen if you have data-driven entities that represent individual
processes rather than game-world objects.

There’s also one thing to be wary of when it comes to vectorization. Nearly all
vectorized floating-point math is not cross-platform deterministic. If you need
deterministic floating-point math, vectorization is probably not the answer.
Integers are fine.

Once you’ve identified some code you wish to vectorize, the next step is to
choose from one of three ways to express vectorized code. Yes, there are three
ways Burst will recognize, and they each have their own tradeoffs.

## Vector Math Vectorization

Vector Math Vectorization is where you use the Unity Mathematics package not to
do vector math, but to express 4-wide vector operations using `float4`, `int4`,
and `bool4`.

Often, you load data using `NativeArray.Reinterpret()` to convert to the
four-wide type, and then do your operations. However, usually Burst will
understand if you just use the type constructors and pass in 4 adjacent
elements.

Once your data is in these wide types, you can use any method in the Unity
Mathematics package that works with them to do various vectorized operations. If
the operator or method uses the 4-wide type as input or output, Burst will
probably generate vectorized instructions for it.

Because you are using explicit types, it is easy to ensure you are getting
vectorized instructions at compile-time. You can interleave it with other scalar
code without worry. And such implementations are also cross-platform, working
across various processors with very different vectorization capabilities.
However, there are some gotchas.

First, some of the methods in the Mathematics package are only fast on some
hardware. For example, `math.bitmask()` is very fast on x86, but is usually not
what you want to do on ARM. On the contrary, ARM has fast `math.csum()`,
`math.cmin()`, and `math.cmax()` for both integer and floating point while x86
usually needs to do some data shuffling to accomplish these tasks.

Second, Burst doesn’t know what the non-vectorized code is supposed to be, so it
won’t tell you whether your vectorized implementation is faster, slower, or
makes no difference (the last of these is very common). You’ll need to profile
for yourself.

Third, this approach doesn’t work with smaller data types like `half` and
`short`.

And lastly, this approach isn’t going to net you all the optimizations Burst has
to offer, especially on modern x86 processors supporting AVX instructions.

If you really like this approach, you can mitigate some of these issues using
the [MaxMath](https://github.com/MrUnbelievable92/MaxMath) library which extends
this style of programming.

It is worth noting that most vectorization Unity uses in their own packages come
in this form. Unity Physics makes heavy use of this style in its broadphase and
midphase algorithms, and Entities Graphics uses this style for frustum culling.

## LLVM Vectorization

Under the hood, Burst uses LLVM for much of its optimization, and therefore much
of the rules around LLVM vectorization also apply to Burst. [The LLVM
documentation](https://llvm.org/docs/Vectorizers.html) provides useful info as
to what vectorization is possible. However, there’s a few extras that Burst
provides.

Nearly all methods in the Mathematics package that operate with a single-width
(`float` instead of `float4`) are compatible with LLVM vectorization. Also,
Burst has special methods you can call that will make Burst throw errors when
LLVM fails to vectorize the code. It is best to consult the [Burst
documentation](https://docs.unity3d.com/Packages/com.unity.burst@1.8/manual/optimization-loop-vectorization.html)
on this.

There are a couple of things to watch out for though. Usage of vector math types
such as `float3` inside your loop will often break LLVM vectorization.
Additionally, Burst doesn’t use the latest LLVM, and so it misses some
optimizations, especially regarding handling of early-outs.

LLVM vectorization comes with plenty of perks, such as unrolling optimizations,
support for full AVX2, validating the vectorized result is not obviously slower
(to the best of its ability), and even vectorizing some suboptimal inputs and
algorithms. However, it can also be much more challenging to write code for.

Burst exposes the LLVM vectorization failure errors in the Burst inspector,
under the *LLVM IR Optimization Diagnostics* tab. This will show some various
errors for different loops as to why LLVM failed to vectorize. When reading
these errors, don’t think of them as “what did I do wrong?”, but instead think
of them as “what about this code is confusing the compiler?” Sometimes adding
local variables, moving some calculations to separate loops, using various Burst
hint intrinsics like `Assume()`, using `FloatMode.Fast`, or even just manually
inlining some code can help the compiler out.

Most resources you find on LLVM vectorization show use cases where you do the
same small sets of operations on some large arrays. However, there’s a
completely different way to use LLVM vectorization that doesn’t often get
discussed.

Instead of working with large arrays, use small `Span`s or `fixed` arrays of a
small power of 2 length (such as 4, 8, or 32). Then, hardcode the loop variables
like this:

```csharp
for (int i = 0; i < 8; i++)
```

If the iteration count is small enough, Burst will completely eliminate the loop
altogether, and replace it with a single sequence of vector instructions. This
is especially fast, because it gives Burst full capability to reason about the
various CPU resources available for each type of CPU and generate highly
optimized code for each. You can then make lots of small loops in sequence, and
let Burst optimize as many of them as it can.

LLVM vectorization breaks by default if you try to call one of your own methods
inside a loop. However, you can address this by either manually inlining your
method, or by using `[Unity.Burst.CompilerServices.Spmd]` enabled by the
scripting define UNITY_BURST_EXPERIMENTAL_SPMD_ATTRIBUTE.

## CPU-Specific Intrinsics

While the previous two methods were cross-platform, sometimes Burst just won’t
generate the code you want it to generate. You might be aware of specific
instructions on various CPUs that are the perfect fit for your problem, and you
want to explicitly tell Burst to use those instructions.

That’s what intrinsics are for.

This is manual mode. You need to wrap each targeted set of instructions in an
`if` block that Burst will optimize out. You’ll need to use `v64`, `v128`, and
`v256` types. And you’ll need to pay close attention to both the Burst docs and
probably some external references. However, aside from some additional load
instructions (which Burst can sometimes be really dumb with), Burst will
generate exactly what you ask it.

## Adjusting Your Algorithms for Vectorization

Most likely, your non-vectorized algorithm is probably doing some things that
make it especially hard to vectorize. It can be a journey to refactor the code
to make it more vectorization-friendly. But stick with it, because very often,
the changes made to assist with vectorization can also speed up the
non-vectorized implementation as well.

The first place to start is if you have vector math types, you will want to
decompose them. This could be by transposing them (next section), or it could be
assigning them to scalar local variables.

Second, if you have branches, try to replace them with `math.select()`. Also, if
you aren’t making use of `math.min()` and `math.max()`, these are your friends.
`math.chgsign()` is also a useful one to keep in mind.

Third, if you have boundary conditions at the ends of your list of elements to
process, consider splitting the logic up such that you only try to vectorize the
range of elements that aren’t affected by the boundary conditions. Then, you can
use

Fourth, if you have any kinds of switch cases or switch expressions, or any
other kind of lookup table, you are going to want to replace these with math.
Analyze these closely, and see if you can come up with a small formula for
calculating these values on the fly instead. Don’t forget you can use
`math.select()` to help define a piecewise expression.

Fifth, if you have random accesses, try to gather them at once in a preliminary
loop. Assuming these accesses don’t depend on each other, the CPU will not wait
on these random accesses one-at-a-time. So this by itself is often an unexpected
optimization.

And lastly, if you have some variable which depends on a previous iteration of
the loop, try to make it so that it only depends on the loop counter instead.

## Transposition

Unfortunately, your input and output data is probably in an array-of-structs
format (AoS), and vectorization really wants struct-of-arrays format (SoA). The
process of converting between these formats is sometimes referred to as
**transposing** the data. This term is especially common for small arrays of
only a few elements.

For use cases where you are dealing with greater than O(n) complexity, the cost
of transposing your data before and after your main algorithm is likely
negligible. However, when your use case involves brute-forcing through large
amounts of data (or just small batches in a hot context), the question of
whether to transpose may not be so obvious. Profiling is the best answer, but
there’s a couple of useful guidelines that can help you.

If your algorithm involves trigonometry, logarithms, or any other intensive math
function on each element, transposing is definitely worth it. Go ahead and make
some utility methods to transpose 2, 3, and 4 quaternions, so that you can then
write a vectorized slerp implementation for them. It is going to be worth it.

For other use cases, it largely depends on how much arithmetic you need to
perform. A rough reference point is that transposing 4 quaternions to normalize
them is near the break-even point (vectorization is advantageous with a really
good AVX implementation, but it is hard to convince Burst to actually output
this).

## Vectorization in Practice

Now that we’ve covered the various ways to express vectorized code, you might be
wondering which method I personally prefer.

The answer is that I use them all at various parts.

I often use vector math vectorization to brute-force through spatial queries
against the edge segments of shapes in Psyshock. Though I use this style in a
few other places as well.

For LLVM vectorization, I think my best example use case is for approximating
the length of Bezier curves. To do this, I evaluate the curve many times to
create 32 segments, which I then compute the lengths of. With AVX, this
vectorizes to less than 8 instructions per segment, which is kinda mindblowing.
I’ve also experimented with small loops in an ECS context with a prototype
tweening system. Burst also sometimes finds loops to vectorize for my audio
system on its own. I haven’t really invested in trying to help it yet. Audio is
a rare use case where the probability of vectorization happening automatically
is a bit higher.

So far, I’ve only broken out the intrinsics in situations where I need to
perform multiple vector comparison operations, and then boil that down into a
single integer value or boolean. There’s not really a good way to express these
with the LLVM vectorizer, and I needed more than what Unity Mathematics had to
offer.

But aside from those use cases, vectorization simply isn’t a tool that I reach
for that often. However, much of that is because so much of my code is optimized
in other ways that I am already hitting the memory bandwidth limit in most use
cases. But vectorization is always something I consider whenever I come across a
new hotspot in a project.

Hopefully this was a useful starting place to getting into vectorization. The
best path forward from here is to work through some real-world examples. Feel
free to share the hotspots you’ve encountered in your own projects on the
community discord!
