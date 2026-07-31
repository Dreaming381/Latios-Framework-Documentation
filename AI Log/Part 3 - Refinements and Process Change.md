# AI Log: Refinements and Process Change

Originally, my plan was to keep the parts AI is involved with separate from
other parts, and log everything done with AI. Having worked through more things
with the AI to improve-upon and land the features from the last two articles, I
have come to the realization that this isn’t the way forward. First, let me
discuss the follow-up projects and their lessons, and then I will discuss the
way forward.

## The Source Generator Situation

Originally, I set out to create the `ILatiosApi` source generator. However, this
was only the first piece of the puzzle. The second piece was creating
`IInjectable`, which is the part that eliminates a lot more boilerplate code. My
Mission.md for this was much simpler than the initial source generator project.
I have learned that Claude is pretty good at extracting requirements from the
XML docs I wrote for the API. And it is also really good at finding files and
types in the project, and following their relationships. I’ve learned I really
only need to be specific about decisions.

With that said, Claude doesn’t write bug-free code. The implementation it
created for `IInjectable` worked at first, but as I ported over some Kinemation
systems, it broke. It was weird. The source generators simply weren’t running,
and I didn’t actually know why. I asked Claude about it, and it somehow
determined through deep Unity editor logs and other files that the generator was
crashing due to a bug, and Unity was disabling it rather than logging any kind
of exception (very helpful Unity). Anyways, it figured out that having multiple
systems in the same file sharing a job name caused problems. I’m sure I will
discover more gotchas like this. But the good thing is that Claude is pretty
good at coming up with fixes for such things. Even still, this lack of
thoroughness is something I have to watch out for.

Once that was resolved, I continued to refactor some more framework systems.
Then I decided that this was time-consuming, so I asked Claude to use my
refactors as examples and attempt to do the remaining systems it felt confident
about, and then list the ones it skipped. All I gave Claude were examples. I
didn’t tell it much else. Claude figured out the pattern and knocked out most of
the systems flawlessly. There were a few systems it got tripped up on and
decided to leave alone. It was understandable how those systems defied some of
the patterns established. The takeaway is that AI is very good at
pattern-recognition, and that can be leveraged to save a lot of time writing
prompts.

## Calligraphics Refactor

After Claude got Calligraphics back into a working state, I left it alone for a
little bit before trying to manually integrate its changes back into LSSS. That
involved copying and pasting parts Claude generated, and then modifying things
to my preferences.

I did this first and foremost to understand Claude’s solution. Once I understood
what a variable did, I would rename it to something that made more sense to me.
I found I had to shorten Claude’s comments, because Claude uses very verbose
comments describing the history of the solution, when I want comments to only
reflect the current state unless there is a known limitation. Claude preserved
some idiosyncrasies that I deleted. It left behind stale comments and unused
variables. It failed to identify things it could simplify. And so I cleaned
things up. I refactored some things. I found ways to simplify the solution
further and remove more tech debt. And after all that, I still had a working
solution.

The lesson here is that Claude is very good at getting things into a working
state. But it sometimes requires further direction or manual cleanup to get
things to a level of quality I am happy with.

## AclUnity for iOS

I know AI has a pretty good track record with build systems, so I asked Claude
to add iOS support to AclUnity using GitHub actions. Claude asked about whether
I wanted the library to be static or dynamic, and what minimum version of iOS to
support (it even suggested versions based on Unity minimum versions). It’s plan
proposed it commit and push the branch and query GitHub. After making the
changes, it just stopped altogether with a comment it was time to push the
changes and see if the artifacts contained the correct contents. So I committed
and pushed to a branch. Sure enough, I got artifacts first try. Does the plugin
binary actually work on iOS? I’m not sure. But in only a few minutes, I can give
something to the community to try, which is a lot nicer than waiting for someone
in the community with the know-how to make a PR for it.

## The Skinned Mesh Crash

This one wasn’t actually from my own use of Claude, but was a pull request from
a team that had encountered some nasty issues due to taking the less-tested
lifecycle path of Kinemation’s skinned meshes. Claude had successfully
identified the main lifecycle cause, which happened to be a bug that my personal
Claude instance flagged during the refactor request. However, it continued to
address issues about buffer allocation integer overflow and other problems that
was causing graphics driver crashes. And while it flagged a bunch of ugly areas,
its proposed fixes for these areas were only treating symptoms. Claude
completely missed that the reason such large buffer allocations were being
requested in the first place was because a `NativeReference` was allocated
uninitialized, and the job designed to populate it was throwing an exception due
to the lifecycle problems. And then the main thread did not recognize the
exception, and just used the uninitialized values.

AI ain’t takin’ my job anytime soon.

## Takeaways

What I’ve learned from all of this is that AI is good at a few things, but also
has some caveats. It is good at discovering and flagging things that have a high
probability of relevance. So while it might not fully understand bugs, it can
bring my attention to areas that need it. It is also good at investigative work.
It can create a working reference implementation. It can crack hard problems
open. It has great pattern recognition, that allows it to work through tedious
tasks asynchronously.

But it is also a bit messy and sloppy, and doesn’t always get things right. It
can’t be fully trusted. It needs the guidance of an expert, and a bit of
trial-and-error. I’m usually opposed to trial-and-error workflows, but at least
with AI, it is largely an asynchronous process.

Ultimately, it is a tool. It is a weird tool. And it is used to save time on
tasks by working asynchronously and discovering/flagging things of relevance.
Its output isn’t good enough on its own. It needs my influence to reach the
quality standard I desire. But it can speed up a variety of areas of
development. I want this to be a tool I can reach for whenever I sense it could
help, not a tool dedicated to specific parts of the framework ecosystem.

Therefore, I’m no longer going to keep logs of all the tasks I have it do, and
instead only log if something new comes up I feel like writing about. I plan to
use it for any kind of code. But I will also make an effort to ensure the
codebase remains one I can support and stand by, even if AI were to suddenly
disappear again.

Thanks for following me on this journey!
