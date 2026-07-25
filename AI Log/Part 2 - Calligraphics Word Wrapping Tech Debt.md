# AI Log: Calligraphics Word Wrapping Tech Debt

Calligraphics has an issue with word wrapping tech debt. I’ve heard that AI has
a very strong understanding of text rendering stacks. So I decided to challenge
AI with a fairly non-trivial problem in a fragile part of the Calligraphics
codebase. This is the saga that unfolded.

## The Mission

When Calligraphics was first created, it only supported basic Latin text, with
only a small subset of rich text tags. The conversion from input string to
`RenderGlyph`s all happened in a single job. Then Fribur came in and reworked
the whole pipeline to use HarfBuzz. This was a powerful upgrade, but due to how
HarfBuzz handles glyph extents, required that the transformation process be
broken up into three distinct phases: shaping, extents calculation, and
`RenderGlyph` generation.

In HarfBuzz, shaping is the process of laying out glyph quads based on kerning,
diacritics, and other fancy script rules. However, HarfBuzz assumes that the
glyphs are all on one giant line. Word-wrapping, paragraphs, and alignment all
need to be handled by the user of the library. Normally, this logic lives right
alongside shaping, with the possibility to re-invoke shaping on parts that got
broken up by word-wrapping. In Calligraphics, the logic was left behind in
`RenderGlyph` generation.

If we want to further improve Calligraphics rendering capabilities with
auto-sizing, right-to-left text, and other features, then we need to get this
word wrapping logic moved into the shaping phase.

This time I have Unity CLI set up, and I already had a separate session validate
that Claude could use it, so let’s see what happens!

## Planning

Claude’s initial plan validated my instructions regarding preserving the 3 phase
pipeline, and reasoned for itself as to why that was a requirement.

Then it discovered that justified alignment was completely broken…

… it was correct.

I have no idea when or how that broke, but I confirmed it was completely messed
up. I probably missed porting a patch I made to TextMeshDOTS.

Claude was also very concerned about the asymmetry of the vertical line
offsetting logic. While planning, it asked whether I wanted to preserve these
issues as-is, or fix them first. Neither was what I wanted, so I picked the
preserve option since that is what it recommended.

The plan it came up with involved generating metadata in a new `NativeStream`
for line wrapping and justification info, and then using that to place the
glyphs in the final phase. I commented that it should instead consider encoding
line wrapping and such inside the `xAdvance` and `yAdvance` fields of each
`OTFGlyph` (the intermediate structure between shaping and `RenderGlyph`
generation). I also commented that preserving behavior wasn’t a hard
requirement. If it was more convenient to fix a bug in the final solution, fix
it then.

Claude’s second plan impressed me. It perfectly acknowledged my requests, and
even came up with a “pen” analogy for how the xAdvance and yAdvance would work.
I approved this plan.

## Offroad Developing

After I approved the plan, Claude made a checklist, and already made one big
mistake. The plan called for building a validation scene first that it could
compare against. However, all validation items in the checklist were placed
after the refactoring work. Claude would be flying blind.

The actual refactoring work was uneventful. Claude just churned away at it while
I ate my food. But then Claude reached a point where it wanted to validate its
work, and that’s where things got interesting.

I had instructed Claude in the first plan that in order to validate things, it
would need to set up a bootstrap, and I pointed it to the templates. Claude
instead found the Calligraphics test project and copied the bootstrap from that.

Claude couldn’t figure out how to import the shader samples. And it decided to
copy the Arial font from the OS to use for testing. It made an ECS test scene,
and took a screenshot of a bunch of white evenly-spaced boxes. Realizing that
was not useful at all, it did most of its debugging by inspecting the
`RenderGlyph` buffers. Surprisingly, that got it pretty far. Then it stopped for
me to review its progress, before continuing to investigate an issue it had
found.

This is when I stepped in and installed the sample shaders for it to use. I told
it I did that, and then it altered its test to create a material and use the
shader before continuing. It was able to track down what it was concerned with,
and this time the white boxes were of different shapes and sizes, and somewhat
spread out. But they were still white boxes. So when it stopped again, I
investigated.

Alpha clipping wasn’t enabled. I turned that on, and there was text! But the
entities were all overlayed on top of each other, and the text ran off the right
of the screen. So I asked it to clean up the test scene and make everything
visible, which it did. And then it realized it had a few more bugs from the
visuals, including the final line not being offset (remember how it didn’t like
asymmetry?). And so I let it work out each bug one-by-one (it would always pause
after each one for me to review). Final line wrapping, centering of text, and
even justification all got fixed. And then I threw in a rich-text heavy string
from the Calligraphics test project. Surprisingly, only two things were broken:
subscripts, and `voffset` downward not pushing the line below down. The latter
it fixed without much effort. For the former, it complained the font didn’t
support it, and that it would write code to emulate the effect. It claimed it
succeeded. It did not.

That was the first real false claim it made about what it did.

So I brought in the fonts from the Calligraphics test project, and sure enough,
subscripts worked fine.

## Takeaways

It wasn’t until this point where I realized that I was actually running Claude 5
Sonnet the whole time. That explains why my credit usage wasn’t problematic, and
how I could let Claude run for a while on a hard problem. Would Fable have
gotten the job done faster? Maybe.

But how fast it gets it done doesn’t matter to me. The key is that it is working
asynchronously. Already, it is moving faster than I can feed it designs.

I’ll have to clean up the code, as Claude left a lot of lengthy comments
following the bugs it created and fixed. But I at least have a working version
of this refactor, a known good state that will help me land this improvement
properly!

Now I know that I can throw fairly hard problems at Claude and let it spin.
Though I should probably set up my old workstation to do that, because play mode
is annoying when I’m not sleeping.
