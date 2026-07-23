# AI Log: ILatiosApi Source Generator

It is time to kick off our adventures into using AI to do time-consuming things
for us. This adventure is focused on implementing a mechanism that will reduce
the amount of code that programmers have to type, or as programmers refer to it,
reducing boilerplate. We’ll be throwing LLMs at this problem, and hopefully end
up with something that users of the framework can enjoy.

## Getting Started with the First AI

Since the last entry, GitHub Copilot decided to raise its prices drastically, to
the point it became unusable. Unfortunately, that’s the only AI harness that
Visual Studio supports. Everything else uses VS Code or other IDEs. So the first
step is to install VS Code and set it up for Unity development. I only intend to
use this for AI, not make it my daily editing experience.

The second step is to add an AI. I’ve heard a lot of good things about
Anthropic’s Claude Fable for the type of Unity development I plan to do, and
have also seen it used to good extent by others I know personally. So that’s
where I am going to start. Claude desktop sign-in failed, but the sign-in into
the VS Code extension worked. That’s probably for the best.

The third step is to set up a dedicated workspace for agents to work on
framework ecosystem things, with risking compromising existing projects. For
this, I created a dedicated Unity project, with copies of both the framework and
add-ons package. I also copied the source generator .NET project into a folder
named SourceGenerators\~ within the Unity project.

Next, I want to document things in a way so that the AI has some initial
direction and I don’t have to repeat myself. So first, I will create an
AGENTS.md file and provide a few words describing the main directories and some
other things to know. I expect that I will evolve this over time. Next, I will
create a Mission.md file, which contains a fairly detailed description of the
task that I want the AI to work on. I’ll commit these to git.

And then, I set the mode in the Claude Code plugin in VS Code to *Plan* and then
type this request:

>   Read Mission.md and create a plan to complete this task.

## What is the Mission?

My goal for this first source generator is to replace some of the functionality
of Unity’s `SystemAPI`. `SystemAPI` has two fundamental flaws:

1.  It is not extensible to custom types
2.  It rewrites the methods, and then uses CECIL to replace your original code,
    meaning when you debug, you aren’t debugging the code you actually wrote

Right now, I am targeting the `Get*()` methods which return type handles and
lookups, as well as the `Time` methods. The former requires caching the handles
in the system inside an `OnCreate()` method. And then retrieving them later
inside `OnUpdate()`.

I’ve already designed the public API by defining an `ILatiosApi` interface. This
interface has an extension method `GetApi()` that returns a struct containing
the actual user API surface. The user creates this struct at the beginning of
`OnUpdate()`. There’s a second method `OnCreateForLatios()` which the user has
to call inside `OnCreate()`, since I don’t want to dig into ILPP for injecting
that into the `OnCreateForCompiler()` calls. Instead, I’ll make this small
constant boilerplate part of the `ISystem` template.

While the `GetApi()` call provides the API surface, `ILatiosApi`’s methods are
the backend implementations that the source generator is responsible for
generating. The methods contain default implementations that throw exceptions.
That allows the code to still compile even before the source generator runs,
which makes for a nice editor experience.

My goal is to have the AI write the Roslyn source generator piece. I’ve written
Roslyn source generators before, including one that adds Burst-compatible
polymorphism that works across assemblies. So I know I can solve this problem
myself. However, sifting through the massive class hierarchy of `Syntax` and
`Symbol` types is quite annoying and time consuming. I suspect an AI can do it a
lot faster.

Inside Mission.md, I introduced the problem to be solved about replacing
`SystemAPI`. I then wrote a section highlighting the files that are important to
the problem, as well as highlighting where the source generator should go. After
that, I wrote four paragraphs, breaking down some of the requirements. I
explained that the source generator needed to find types implementing
`ILatiosApi`, and scan the implementation for calls to the API methods. I
specified the source generator should create a new partial definition, and
specified how it should create the cached variables. I explained how it should
initialize the variables in the `__OnCreateForLatios()` implementation. And I
explained the strategy for the other methods to utilize `typeof()` comparisons
and `UnsafeUtility.As<>()` to select the right field and return it as type `T`
inside the generic implementation.

It took me about 30 minutes to articulate all of this. With my 80 character per
line limit in markdown, the entire Mission was 50 lines long, so not too much. I
was trying to be a bit precise to guide it.

## The Plan

Claude processed the request for several minutes. I could have easily switched
over to some physics code while it processed (which is my desired workflow for
AI), but I was too curious. So instead, I watched it identify and read files,
validate assumptions, and burn several thousand tokens thinking. And then it
spit out a plan. Here’s a snippet of what it assessed:

>   This generator is a new kind of problem for this codebase: existing
>   generators emit fixed boilerplate once per matching type declaration. This
>   one must additionally **scan the body of the matching type for
>   LatiosApiInvoker\<T\>.Get\* call sites**, infer field types/count from
>   usage, and emit a per-usage-shaped \__LatiosApiState cache struct. The goal
>   is a design that reuses the established project structure/helpers but adds
>   the semantic-extraction logic needed for usage-driven codegen.

So, that statement about fixed boilerplate applies to some generators, but not
all. However, it correctly assessed this is a new type of problem, and why.
Prior to that snippet, it identified key points of the architecture already
in-place for the Source Generators project. And then it went on to define its
implementation strategy.

The plan identified all the files it would create, and what it would add to
them, outlining the full algorithm for finding what fields to generate, and how
to aggregate them. It also identified which files it believed could be left
alone because they were sufficient as-is. It then outlined how it would test its
work both using the source generator test project, and it also was bold enough
to suggest building the generator dll, copying it into the local framework
version, and then building a test directly in the project. (Note: I gave it
permission in the Agents.md file that it could do whatever it wants in the
Assets folder to validate.)

I suspect it will run into a little bit of trouble with validation, because I
didn’t add a bootstrap to the project yet. And I also don’t have Unity CLI
hooked up yet to let it run with. But I want to see it try anyways.

Before that, let me check how many tokens I’ve used…

9%

Now, that’s with a 50% new subscriber promotion, but even still. For 5 hour
windows, that is very reasonable for the pace I want AI to work. And I think I
have enough to tell Claude to try and execute the plan. So I’m going to do that,
and then go to sleep. I’ll let you know what happens when I wake up.

## The 1st Attempt

The next morning, I woke up with Claude waiting on me to approve it to run build
commands. So after approving a few things, it finally got stuck trying to
validate Unity, but only because I already had the project open. It otherwise
tried opening the project in batch-mode.

Did it work?

Mostly. Enough that I consider it quite successful.

There were a few issues it left behind. Apparently Unity’s own source generators
add the `[CompilerGenerated]` attribute on every system, so the one added by
this generator produced errors. Also, I want it to use explicit interface
implementations rather than implicit when implementing `ILatiosApi`. Though I
never told it that. I also never told it to try and group values by the `bool`
constant, so it always used that as the second check in every `if` type check.

Looking deeper, it also messed up the initialization of `LatiosWorldUnmanaged`,
because it used an extension method without including the namespace. I should
have made a `StaticAPI` method for that, so that’s partly on me. And it also
tried to handle cases where users created extension methods against
`LatiosApiInvoker<>`. It seems to have gotten confused by my goal to make this
API extensible. It is already extensible via the `Get<>()` methods. I also
noticed it was recreating a `StringBuilder` with each update. I’d like to pool
that.

In the future, I will likely ask Claude to fix these things. But for this first
time, I am going to clean it up myself to prove to myself that I am able to
maintain the codebase.

And so with the exception of the `bool` constant grouping, I made the changes
myself, updated the tests, and built. It only took me a few minutes.

## The Result in Production

The final test was to bring the changes over to the real source generators repo,
build it, and bring it into LSSS. Then, I modified `AutoDestroyExpirablesSystem`
to use the new paradigm.

It works!

The Burst codegen is exactly what I wanted too!

I think the reason this worked so well is because I had a good design in place,
with the API already established. And I was able to articulate with precise
detail what I wanted the source generator to produce.

Is the code Claude created identical to how I would have written it myself even
after my cleanup? No. But if the final state came in as a pull request, I would
absolutely accept it! The code Claude produced is quite precise and concise.

If I exclude setup time, the time spent writing this log, and the time I spent
distracted thinking about what I want to do next, I spent at max 90 minutes on
this problem and have a working solution. This probably would have taken me a
couple days to sort through all of the Roslyn APIs and come up with a solution.

Now I need to get the framework migrated over to this new API!
