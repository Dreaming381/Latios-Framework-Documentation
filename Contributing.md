# Contributing

Oh hi! Awesome of you to stop by!

No. I don’t want your money. And no, I don’t drink coffee.

If you are an artist or someone who is dead-set on using disposable income to
make the Latios Framework better, jump to
[here](#_Disposable_Income_Contributions).

Otherwise, I’m assuming you are here because you are a programmer and want to
contribute to an open source project the way programmers do. Keep reading to get
some ideas.

## Before You Jump In

The Latios Framework is maintained to a very high standard. This doesn’t mean
that fewer bugs sneak through, but rather that when bugs are discovered, fixes
can come quickly and with minimal disruption to the existing API until the next
feature release. I have a design process that helps enforce this. I have never
been able to teach anyone else this process. And consequently, my stance on
contributions directly to the framework is that I’d strongly recommend sticking
to smaller quality-of-life improvements or the areas where help is most wanted.
That’s not to say you can’t try to get a big feature of your own design merged,
but mentally prepare yourself for a tough journey.

However, for the [add-ons
package](https://github.com/Dreaming381/Latios-Framework-Add-Ons), go freaking
nuts! This is by far the best place to contribute to the Latios Framework
ecosystem. You can add your own add-on, or reach out to the primary author of
one of the other add-ons. As long as you follow the inert-by-default rule, you
don’t need to provide any justification. Just go for it!

If you need ideas, feel free to reach out. There are plenty of possible items
across a wide variety of areas and at various levels of difficulty.

Also, the Latios Framework can always use more learning resources such as blogs
and tutorials, or good sample projects showing off various ways to use the
features and APIs.

## Developing Using a Public GitHub Account

If you would like to use git to develop a new feature, follow these steps:

1.  Fork the Latios Framework (or add-ons) from GitHub.
2.  Create a Unity Project where you would like to develop your feature. This
    can be a new or existing project. No one but you will see this project.
3.  Clone or Submodule your fork of the Latios Framework (or add-ons) into the
    Packages folder of your project.
4.  If you use a code formatter and plan to modify existing code, format all
    code files you intend to touch first and commit it. (If your changes are
    small, this may not be important.)
5.  Make your changes and test using your project.
6.  Make as many commits as you like.
7.  Do not worry about documentation or version numbers. Documentation updates
    are nice, but not necessary. Unity Tests are also optional. If you are
    making your own add-on, you may choose to use your own add-on version
    numbers and change them as necessary.
8.  Push your changes to your fork.
9.  Make a pull request. There’s a pull request template that will ask you to
    fill out various details to help further guide the process.
10. PRs will usually land in a staging branch or be merged locally to be pushed
    in the next update. Pay attention to reviews. Some reviews may have
    suggested improvements. However, if such suggestions are trivial, they might
    be made in the next release commit on top of your commits’ changes.

## Developing Without GitHub

If you use some other source control mechanism or an internal git host, follow
these steps:

1.  Create a Unity Project where you would like to develop your feature. This
    can be a new or existing project. No one but you will see this project.
2.  Download, Clone, or Submodule the Latios Framework (or add-ons) into the
    Packages folder of your project.
3.  If you use a code formatter, format all the code in the Latios Framework (or
    add-ons) and save a copy in a zipped folder.
4.  Make your changes and test using your project.
5.  Do not worry about documentation or version numbers. Documentation updates
    are nice, but not necessary. Unity Tests are also optional.
6.  Save a copy of your changes to a zipped folder.
7.  Send your zipped folder (and the zipped folder from step 3 if you have one)
    via Discord, Unity forums PM, email, or any other contact channel you are
    aware of. The zipped folder should typically be under 2 MB in size (exclude
    documentation in your zip).

Your changes will be cherry-picked into a local development project and included
in the release. You will not be included in git metadata associated with the
changes. I use tools like Meld for merging such changes, in case you were
wondering.

## Naming

My naming conventions are a little bit different from Unity and more closely
resemble my C++ naming convention. You don’t have to follow these when
developing new features. But expect deviations to be modified if you make
framework changes or add-ons you are not the primary author of.

-   All data types, enum values, and methods use PascalCase.
-   All fields and properties use camelCase.
-   Private and Internal fields and properties may have an “m_” prefix if they
    are accompanied by public members.
-   Zero-size tag components have a “Tag” suffix.
-   Zero-size enableable components have a “Flag” suffix.
-   ECS systems have a “System” suffix.

If you have a tendency to be overly bland or generic with your type, variable,
and function names, please use comments to provide details so that I can better
understand your code.

## Formatting

In general, do not worry about formatting. Instead, format the files you intend
to modify first, and then make your changes such that I can see the diff of just
the changes and not your formatting.

I have my own tool for formatting which I use for personal development called
“Alina”. It is not perfect, so if you are interested in helping it be better and
usable by more people, let me know!

## Unit Tests

If you want them, write them. But in general, my philosophy has always been to
test things with real projects. That way, I not only test the code itself, but
also the practicality of such APIs. If you don’t like unit tests, don’t worry
about it.

## The Final Step

Let me know what name you would like to be referred to in the Contributors
section of the README. If you don’t have a preference, I will typically default
to your Unity forums or Discord username if I know it, otherwise I’ll use your
GitHub username. If you have a webpage you would like to be linked to your name,
let me know that as well.

## My Own Workflow

I manually synchronize development of the Latios Framework across several
projects. This means I often have multiple versions of the framework with
experimental features. Eventually changes are organized in a release repo which
pushes directly to the official repo on GitHub.

If you would like a copy of any experimental version of the framework,
especially if you wish to develop new features against it and make some of the
existing experimental features release-viable, let me know. But don’t expect
that unreleased experimental stuff to not obliterate your nose with awfulness.

## “I’d love to help, but the stuff you do is way over my head!”

You don’t need to develop new features to help out.

If you use the Latios Framework, you can make demo projects which can help
others understand how to use the API. You can send those projects to me via zip
files or send me a public link to the demo.

If you are an artist or designer, you can create assets and send them to me to
be featured in demos, examples, and tutorials.

Also, a huge help for future users is writing devlogs about your adventures with
the Latios Framework. You can even showcase your progress in the Discord! We
love it!

If there is a particular feature that you want, build a sample project where the
feature could easily be leveraged and share it with me. This gives me a concrete
use case to develop the new feature against and ensure it works. Doing this cuts
development time of the feature in half or more!

## Disposable Income Contributions

If you reached this section because you have disposable income and really,
really want to put it towards the Latios Framework, there are primarily two ways
you can help. Both involve commissioning other individuals of your choice to
contribute.

In the future, I hope to have a community fund for this purpose. But currently
doing so on my own would be a massive time investment. If you would like to be
involved in such an endeavor, whether as a donator, a manager, or a commissioned
recipient, please reach out to me privately.

### Art

Most “free” art isn’t actually free. Places like Mixamo and the Unity Asset
Store do not allow for redistribution of assets in source form. Unfortunately,
this restriction makes these assets unusable in sample projects. Some art is
licensed under Creative Commons Attribution; however, this license is a bit
vague on how to properly satisfy the conditions of attribution. Some art is
given the “free” label, but otherwise does not specify how the art may be used
in any way.

One of the best ways to help out framework development is to provide art assets
that can be used for future experiments and sample projects. You can make them
yourself or commission someone else. If you’d like, you can involve me in the
process to ensure the result will be usable, but this is not a requirement. CC0
or a similar unrestricted license works best.

If you reached this section because you have disposable income and really,
really want to put it towards the Latios Framework, the best thing you can do is
commission an artist to create assets that can be used for future experiments
and sample projects (yes, this does make feature development go faster because I
can better test the effectiveness of features in more realistic scenarios).
There are plenty of sites out there where you can commission an artist. Pick
whoever you like.

If you are an artist, pretend you were commissioned for whatever amount you like
(or don’t if that messes with your mojo). Assume whatever art is contributed
will be released as CC0 (or whatever other license you like that would permit
redistribution) within a year of its creation.

While most art will be useful in some capacity at some point (use common sense),
here’s a living list of ideas that will be especially helpful:

-   3D characters
    -   While there are good options for rigid low-poly characters, characters
        that can make use of physics-based secondary effects such as cloth,
        hair, soft-bodies, or soft IK are very scarce. Characters are an
        important part of the Latios Framework. But the framework can only be as
        good as the art it is built to handle.
-   Environments
    -   Test environments tend to be simple and clean, while real game
        environments are often much messier, which will stress physics and
        rendering systems much harder. Environments that push the limits of the
        tech are awesome! But environments that can be dropped in for sample
        projects are also a great way to get more sample projects out in the
        wild.
-   Weapons and Effects
    -   It is not always obvious how all the different features of the Latios
        Framework can be tied together. Having ready-to-use weapons and effects
        can be the launchpad of better samples to demonstrate multiple features
        working in tandem.
-   Terrain
    -   If you want terrain support with trees and details and all that to work
        in ECS, then make the scene you want to see function in ECS and send it
        our way.

### Samples

The Latios Framework’s APIs are designed to support many, many use cases.
Naturally, I do not have time to build samples for showing off all of them.
Here’s a few ideas for some things you might be able to demonstrate:

-   Psyshock Scenes
    -   Feel free to consider out-of-the-box physics demos using physics engines
        from the Add-Ons package, such as Anna physics.
-   Kinemation Character Customization
    -   This is easy to do with Kinemation. But setting up a complete solution
        with assets and UI integration is something I’ve just never got around
        to.
-   LifeFX VFX
    -   LifeFX allows for all sorts of VFX to be created. I’m not a VFX artist,
        so it would be awesome to have someone who is show off all the things
        that can be done with graphics events, tracked transforms, and even
        Kinemation deform buffer vertex sampling.

## Friendly Development Areas

I tend to focus on code that plays to my strengths as a developer, which usually
involves performance-sensitive code and complex data. Consequently, I tend to
not spend time on things to improve the editor experience or reduce boilerplate.
If you are looking to help, these are the areas that you can easily provide a
lot of value to:

-   Editor scripting and tools
    -   Most of my authoring components use default inspectors and have very
        limited or broken debug tooling. This can be drastically improved.
-   Source Generators
    -   I have plenty of ideas for source generators that can be used to reduce
        boilerplate, but often times, I struggle to justify the time investment.
        If you like the idea of reduced boilerplate, feel free to reach out, and
        I can walk you through some of my ideas.

However, if you are like me and are more interested in the deep technical
runtime code, there are a few of areas I could use your help as well. Reach out
on Discord if you want to get involved in one of these:

-   2D
    -   I don’t intend to develop 2D on my own, but I happen to know a lot about
        it, and am not opposed to guiding individuals or even a full team to
        develop a suite of 2D functionality. I’d even be willing to help with
        some of the challenging aspects of it if needed.
-   Psyshock
    -   Psyshock’s architecture lends itself to being more friendly for
        contributors to help out, that-is if you can wrap your head around the
        math involved. There are so many different constraint solvers, spatial
        tests, and other things that would all be welcome in the ever-growing
        Psyshock toolbox.
-   Calligraphics
    -   Calligraphics is the most community-contributed feature of the
        framework. It turns out that world-space text is critical to many
        applications, even if it isn’t the most exciting. If Calligraphics
        doesn’t quite fully meet your needs, and you want to help close that
        gap, please reach out!

If you have further inquiries, PM me on the Unity forums or DM me on Discord.
