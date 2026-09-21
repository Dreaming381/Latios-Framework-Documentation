# AI Log: A Month of Features and Improvements

It has been over a month since I started using LLMs as part of my development
workflow. I want to take some time to discuss everything that has happened since
the last log.

## What Has AI Accomplished?

A lot actually. Like way more than I ever thought I would have ever gotten it to
succeed with. Here’s the highlights:

### Roslyn Source Generators and Analyzers

What started with `ILatiosApi` has grown into something far more powerful. In
addition to the base APIs, there’s now the `IInjectable` interface for
automatically populating handles in jobs and other structs, as well as
`IJobEach`, which is a full `IJobEntity` replacement with enhanced features and
flexibility. I’ve also introduced Roslyn Analyzers into the mix, so that users
have a better experience in their IDEs.

### Unika Upgrades

Unika got quite a few new features that hopefully make it a lot more accessible.
There’s a `[Needs]` based reflection system for automatically populating
`ComponentBroker`s within a given context. There’s proper debugger properties
for inspecting scripts inside an IDE. Those same properties also drive a new
Scripts ECS runtime inspector tab. And there’s a new automatic authoring
generation algorithm which eliminates all the baking boilerplate, allowing
scripts to serve as fast prototyping tools.

### AclUnity iOS Support

I’m not very good at dev ops tasks, especially for platforms I can’t test
myself. So trying to extend the cmake and GitHub Actions pipeline via trial and
error to produce an iOS build was not something I wanted to spend days on. AI
navigated the requirements confidently and produced a binary artifact on its
first attempt. As of writing, I still don’t know whether that artifact works,
but I’d much rather have it for the community to try than wait for some generous
soul to make a pull request.

### Entity Store V2 Support

ENTITY_STORE_V1 is no longer a requirement. This was tech debt that I knew
exactly how to solve, but was reluctant to spend the time on it. AI did it for
me.

### IJobEach Integrations

To help validate `IJobEach`’s readiness, I had AI help build `ThreadCache` and
`OptimizedSkeletonInputsAspect`. It also built some of the QVVS Transforms
`IParameterHandle` implementations.

### QVVS Transforms Updates

AI has been especially helpful in improving the QVVS Transforms API. It added
combined position and rotation settings across the whole stack, and helped keep
the ticked and non-ticked versions of APIs in-sync.

### Refactoring the Framework

With new APIs such as `IJobEach` and `ChunkEntityBatchEnumerator`, AI helped
refactor the entire framework to use these new APIs. I also had it add
UNITY_DOTS_DEBUG support everywhere.

### Dynamic Allocations for Custom Commands in Command Buffers

`IInstantiateCommand` and `ICustomCommand` only provide a single struct to
define a command. But now, their command buffers can provide memory spans to
store inside them, mimicking the pattern of `PairStream` and Audio ECS pipes.

### Color Space Conversion

There are now a small number of APIs for converted between linear and gamma
color spaces.

### Calci KeyedCurve Updates

Calci’s `KeyedCurve` received new APIs for SIMD sampling and evaluating
integrals in time.

### New Add-Ons

Mash is an add-on for converting Input System action maps into ECS data in a
streamlined way. Ribbons is an add-on for baking `LineRenderer` and
`TrailRenderer` and rendering them using `UniqueMesh`.

Smoothies and Shuriken have both been revived and are still under active
development as of writing. There’s also ShaderWaitress which is an evolving tool
designed specifically for agents to manipulate shader graphs.

### Bugfixes Galore

Especially when developing add-ons, the AI encountered issues which made me
aware of framework bugs. Sometimes it correctly identified the bug itself and
fixed it. I also had it scan through code a few times searching for certain
types of bugs, and to report a list of defects it found. In one scan where I
focused it on one-liners and copy-and-paste bugs, it came back with \~15
defects, none of which were false positives, all of which were my fault. These
are the types of issues that end up biting users and keeping me in the support
loop.

## Regarding the Vibe-Coding Trap

There’s a common trap with AI to just trust it and let it run wild, because it
will usually achieve what you ask for. However, that’s often a trap, because it
only looks like it achieved what you asked, but in reality it decided to skip
something very important. It is very easy to let it take over the codebase, and
that’s not good, because it writes verbose comments for itself and will happily
solve a problem the wrong way. I don’t trust the AI. I trust the code I read and
the resulting behavior. This has been my philosophy in development so far.

If I ask it to develop runtime code in the framework, I review the code very
closely to ensure it meets the same standards I’d hold any other PR. I also try
to fix most framework bugs the AI encounters myself, to ensure I maintain
understanding. I’m still opting to develop the most critical pieces on my own,
or at least establish essential patterns which I can then let the AI follow.

If I have AI develop source generators, I care less about its implementation,
and more that the generated code is what I expect. This level of indirection
saves a lot of review time. Source generators are small very-well defined apps,
well within the realm of what an AI can manage on its own.

Similarly, in Unity, Editor tooling development doesn’t have any real definition
of “correctness”. Instead, developers are meant to simply follow “best
practices” and assume the code is correct until it breaks. That’s a very natural
philosophy for AI, and I’m no better than it when it comes to this style of
development, so that’s another area I let it free.

I’ve also let the AI try its hand at add-ons. I spend quite a bit of time with
it ensuring the design and architecture is reasonable. And when things come up
in the implementation, I decide the tradeoffs. But otherwise, I let AI do all
the work. These are optional add-ons that would have otherwise lived in the
backlog for a very long time. My hope is that the AI will have produced one of
them good enough to help someone. Time will tell.

## Where AI Has Been Strong

AI has helped in a lot more places than I expected.

### Bug Finding

I think the most impressive part is how consistently it finds real bugs. If
you’ve seen what goes into my patch releases, you know that a lot of dumb
mistakes have historically landed into releases uncaught. That’s because the
number of use cases the framework tries to account for is far larger than what I
can test on my own. AI can catch those mistakes and build proper validation
around the fixes while I’m taking care of life things.

### Using Established Patterns and Idioms

AI also does really well with prior art in the code. I refactored a few systems
to use `ILatiosApi`, and asked AI to do as many more that it could trivially. It
had no problem with that kind of task. It has also been good at keeping the
ticked and non-ticked QVVS Transforms APIs in-sync. And whenever I asked it to
develop a new feature, I would point it at specific keywords or files that
contained patterns I wanted it to borrow from. Sometimes, it finds these
patterns on its own.

### Operating Under Requirements

AI also does well with strongly-defined requirements, no matter how technically
complex they are. I’ve found I don’t need to explain concepts to the AI the way
I would a human. It behaves with a lot of domain knowledge. And with good
definition, it rarely misses at producing correct results.

### Understanding Motives

A really surprising one I discovered, is that AI responds very well to motives.
If I describe the motive behind a feature and the requirements I am asking it to
design for, it will do a much better job at navigating hiccups that come up and
steer towards more desirable remedies. This is great for working around engine
bugs or limitations. It is also a shortcut that let’s me spend less time
articulating requirements.

### Investigating Unknowns

Lastly, AI is good at investigative work. Whether that be figuring out how Burst
behaves in various scenarios, or analyzing and comparing architectures,
validating or disproving assumptions, or figuring out what all Unity changed or
added in an update, and whether or not those changes make a particular thing
possible. AI doesn’t always do these things if you don’t specifically request
for them. But when you do request them, it can save a lot of time in developing
an understanding of how things work.

## Where I Have Not Been Successful with AI

Or more specifically, Claude Opus 5 at the time of writing. Anthropic is the
only AI provider I have subscribed to for personal development so far.

### Tunnel Vision

I have come to describe AI models as “experts with sometimes severe tunnel
vision”. Sometimes, they see a bunch of symptoms, and come to the wrong
conclusion prematurely. Sometimes the models knock themselves out of the loop
with pure persistence. Sometimes they get stuck and end up coming back to me
asking how I’d like to proceed. The larger the model, the less likely this is to
occur. I found Sonnet 5 was running into these traps a lot, whereas Opus 5 has a
much better understanding of my development environment and motives. Despite
draining weekly credits faster, it has been overall cheaper at arriving to
satisfactory solutions. Therefore, I stopped using Sonnet 5 for nearly all
tasks. It is a shame though, as Sonnet 5’s writing voice is a little better than
Opus 5’s.

With that said, Opus 5 is not immune to tunnel vision either. It still doesn’t
always recognize when it set up a shader incorrectly, or that play mode paused
on an error stopping animations from progressing. And sometimes, it doesn’t know
rules or details that lead it to the wrong conclusion if I don’t catch it. When
I point things out, it usually gets up-to-speed quickly and doesn’t repeat its
mistake until the next compaction. That’s why I say “tunnel vision”.

### Following 1-Off Directions

This one is goofy. Unity’s pipeline package has a memory leak, and also keeps my
CPU from idling. So I can’t keep the editor all the time. When I want to ask
Claude one last task before leaving or going to sleep, I ask it to close the
editor when it is done that one time. Later, when I reopen the editor and have
it continue, it will close the editor on every turn until I scold it. It is
weird, because it has no problems with other state-machine-like instructions.
For example, I sometimes ask it to wait some number of hours (so that my 5 hour
window resets) before reading a file and performing the task specified. It has
no problem with such commands. But for whatever reason, asking it to close the
editor once because I want to sleep confuses it.

### Assuming Things Prematurely

One of the tasks I’ve assigned Opus 5 has been to replicate Unity’s Particle
System in ECS. Initially, there was a lot of back-and-forth to develop the
architecture. But after that, its task was to reverse-engineer behaviors by
probing the built-in system, reading the documentation, and comparing to the ECS
solution it was developing. I have been pointing out visual issues half the time
for roughly 3 weeks now. It is honestly the experience I was expecting to get
for all problems. All of its bugs come from wrong assumptions which later it
ends up disproving when I point out a prefab comparison that doesn’t match. To
be fair, Unity’s Particle System is super inconsistent, especially between
whether parameters are in degrees or radians. But bad assumptions or forgetting
to implement things are AI’s two biggest bug contributors.

### Documenting for Users

Claude Sonnet 5 and Opus 5 are both horrendous at creating useful documentation.
I have to edit or delete a lot of the comments it leaves behind in code. If it
generates a README for a new add-on, it usually fails to follow the template,
and then creates a bunch of word-vomit. I even tried asking it to focus solely
on adding visuals to the existing documentation, since that’s something I have
historically been poor at. Out of the \~25 visuals it generated, I kept 1, which
was a plot of Myri’s falloff curve. The rest were ugly and not helpful. It also
forgets to keep docs and comments up-to-date in places.

I could probably send an agent into a loop with sub-agents to develop a skill
that eventually resolves a good amount of this. But right now, I would rather
spend tokens on add-on development. I’m hoping other models may be able to fill
this void and allow for real learning resource development.

### Eating Up My Time

Reviewing the output of AI takes a lot of time, and this often makes it
difficult for me to focus on other tasks. I’m starting to learn that I need to
make AI work more while I am doing life things and not have it run when I am
trying to code things myself.

## Final Thoughts

AI is a tool. It has its strengths. It has its weaknesses. I believe I am using
it to make the framework better.

Over the last month and a half, AI has greatly improved the ergonomics of the
framework, covering for the usability regressions that came with Unity changes
and the reaction to those changes in the framework. It has helped me break
barriers I felt stuck behind. And it has found a lot of my stupid bugs.

These improvements are designed to help developers make better games faster,
regardless of whether the developers use AI. And the framework itself makes DOTS
more accessible, which means more efficient processing of games, and less
electricity used by players. Care and caution are needed. There’s a lot of
people in the world who probably shouldn’t use AI. But I also believe there are
people who can wield AI in truly good ways. I hope I’m one of those people.

Thanks for reading!
