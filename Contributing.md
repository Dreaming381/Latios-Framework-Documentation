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
feature release. Because of this, I simply cannot accept any and all
pull-requests. So please be ready for criticism and discussion before trying to
contribute a big new feature.

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

Every once in while, I get asked: “Why don’t you use a development branch and
release branch like other GitHub projects?”

The answer is because I develop and test the framework inside other Unity
projects, often version-controlled. Do you know what each of these commands do?

```bash
git submodule update --init --recursive
git merge --ff-only pr/381
git rebase -i HEAD~3
```

If any of the above were unfamiliar to you, then that’s why. I don’t want to
have to educate people on how to use git to be able to contribute to framework
development or experiment with the sample projects.

Instead, I manually synchronize development of the Latios Framework across
several git projects using Meld. This means I often have multiple versions of
the framework with experimental features. Eventually changes are organized in a
release repo which pushes directly to the official repo on GitHub.

If you would like a copy of any experimental version of the framework,
especially if you wish to develop new features against it and make some of the
existing experimental features release-viable, let me know. But don’t expect
that unreleased experimental stuff to not obliterate your nose with awfulness.

## “I’d love to help, but the stuff you do is way over my head!”

You don’t need to develop new features to help out.

Us the framework to write stuff, make stuff, and share stuff. Anything helps,
whether it be blog of misadventures, a cute little prototype, a tutorial video,
some test assets you’d like to see in a future sample, anything really.

You can also collaborate by building the test project for a new feature, and
letting me build the feature inside the project. This speeds up development more
than you think, and it is a good way to prioritize the feature you want sooner.

## Non-Code Contributions

If you reached this section because you don’t want to write code, but you do
want to help, first off, thank you!

There’s two areas you can help me out: Art and Connections.

### Art

I could really use the help of artists and level designers to make assets and
pre-assembled environments and effects that I can use is demos, samples, tests,
and benchmarks. Most assets online don’t allow me to distribute the full project
as source if the project includes the asset. So anything created needs to be
licensed so that I can do that. CC0 is really good for that.

### Connections

If there is someone you would like me to know, feel free to introduce them to
me. And if there’s a community space or website you would like me to be familiar
with, feel free to point those to me. I’m not looking for research papers
though.

If you have further inquiries, PM me on the Unity forums or DM me on Discord.
