# Contributing

Oh hi! Awesome of you to stop by!

No. I don’t want your money. And no, I don’t drink coffee.

If you are an artist or someone who is dead-set on using disposable income to make the Latios Framework better, jump to [here](#_Disposable_Income_Contributions).

Otherwise, I’m assuming you are here because you are a programmer and want to contribute to an open source project the way programmers do. Keep reading to get some ideas.

## Picking a Feature

If you are just looking to contribute some bug fixes, skip this step.

If you already have a feature in mind, be sure to run it by me via various channels first to increase the chances of your pull request being accepted. The Discord server is usually the best (public or DMs) if you want fast responses. But forum PMs, GitHub discussions, and email all work as well.

If you don’t have a feature in mind, worry not. There are plenty of open challenges to solve. Not all of them require the brainpower of a super-genius either. Convenience utilities, Editor scripting and tooling, and dev-ops are all areas that could really use your help. And someone needs to make the example projects.

If you a looking for an optimization challenge, there’s plenty around too, many of which will have real impacts for lots of users.

You can find open tasks [here](https://github.com/Dreaming381/Latios-Framework/issues?q=is%3Aopen+is%3Aissue+label%3A%22open+to+contributors%22). If you don’t see a task in an area that interests you, reach out via Discord, forums, GitHub, or email to see if there are other items that are less well-defined but are just as badly needed.

## Developing Using a Public GitHub Account

If you would like to use git to develop a new feature, follow these steps:

1.  Fork the Latios Framework from GitHub.
2.  Create a Unity Project where you would like to develop your feature. This can be a new or existing project. No one but you will see this project.
3.  Clone or Submodule your fork of the Latios Framework into the Packages folder of your project.
4.  If you use a code formatter, format all code in the Latios Framework and commit it. (If you know in advance which files you need to modify, you can choose to only format those files instead. But make sure you do not commit any other formatted files in future commits.)
5.  Make your changes and test using your project.
6.  Make as many commits as you like.
7.  Do not worry about documentation or version numbers. Documentation updates are nice, but not necessary. Unity Tests are also optional.
8.  Push your changes to your fork.
9.  Make a pull request. There’s a pull request template that will ask you to fill out various details to help further guide the process.
10. PRs will land in a staging branch. Pay attention to reviews. Some may have suggested improvements. However, if such suggestions are trivial, they might be made in the next release commit on top of your commits’ changes.

## Developing Without GitHub

If you use some other source control mechanism or an internal git host, follow these steps:

1.  Create a Unity Project where you would like to develop your feature. This can be a new or existing project. No one but you will see this project.
2.  Download, Clone, or Submodule the Latios Framework into the Packages folder of your project.
3.  If you use a code formatter, format all the code in the Latios Framework and save a copy in a zipped folder.
4.  Make your changes and test using your project.
5.  Do not worry about documentation or version numbers. Documentation updates are nice, but not necessary. Unity Tests are also optional.
6.  Save a copy of your changes to a zipped folder.
7.  Send your zipped folder (and the zipped folder from step 3 if you have one) via Discord, Unity forums PM, email, or any other contact channel you are aware of. The zipped folder should typically be under 2 MB in size (exclude documentation in your zip).

Your changes will be cherry-picked into a local development project and included in the release. You will not be included in git metadata associated with the changes. I use tools like Meld for merging such changes, in case you were wondering.

## Naming

My naming conventions are a little bit different from Unity and more closely resemble my C++ naming convention. You don’t have to follow these when developing new features. But expect deviations to be modified in an official release.

-   All data types, enum values, and methods use PascalCase.
-   All fields and properties use camelCase.
-   Private and Internal fields and properties may have an “m_” prefix if they are accompanied by public members.
-   Zero-size tag components have a “Tag” suffix.
-   Zero-size enableable components have a “Flag” suffix.
-   ECS systems have a “System” suffix.

If you have a tendency to be overly bland or generic with your type, variable, and function names, please use comments to provide details so that I can better understand your code.

## Formatting

In general, do not worry about formatting. Instead, format the files you intend to modify first, and then make your changes such that I can see the diff of just the changes and not your formatting.

I have my own tool for formatting which I use for personal development called “Alina”. It is not perfect, so if you are interested in helping it be better and usable by more people, let me know!

## Unit Tests

If you want them, write them. But in general, my philosophy has always been to test things with real projects. That way, I not only test the code itself, but also the practicality of such APIs. If you don’t like unit tests, don’t worry about it.

## The Final Step

Let me know what name you would like to be referred to in the Contributors section of the README. If you don’t have a preference, I will typically default to your Unity forums or Discord username if I know it, otherwise I’ll use your GitHub username. If you have a webpage you would like to be linked to your name, let me know that as well.

## Large Features Notice

When a member of the community contributes a large feature, it may not always be possible or practical for someone else to fix any bugs related to that feature. For this reason, large features contributed by the community are often marked as “experimental” or “addons”. Bugs in such features may not be fixed in regular patch releases.

With that said, very seldom do people get designs right the first time. Much of the Latios Framework comes from analyzing previous designs and finding ways to do things better. Even if your big feature is experimental, it is still valuable and will help the community learn and grow and eventually do something amazing!

## Consider Open Projects

[Free Parking](https://github.com/Dreaming381/Free-Parking) is an open project designed as an experimental playground for framework developers, users, teachers, learners, experimenters, prototypers, tech demo-ers, open-source enthusiasts, and much more! There, you’ll be able to collaborate quickly with other developers to help land the features you desire as fast as possible.

For something that is a little less formalized, there is also the [Hack-Labs](https://github.com/Dreaming381/Hack-Labs) repository, which is intended to be a contributors-only repository for various developments.

## My Own Workflow

I manually synchronize development of the Latios Framework across several projects. This means I often have multiple versions of the framework with experimental features. Eventually changes are organized in a release repo which pushes directly to the official repo on GitHub.

If you would like a copy of any experimental version of the framework, especially if you wish to develop new features against it and make some of the existing experimental features release-viable, let me know. But don’t expect that unreleased experimental stuff to not obliterate your nose with awfulness.

## “I’d love to help, but the stuff you do is way over my head!”

You don’t need to develop new features to help out.

If you use the Latios Framework, you can make demo projects which can help others understand how to use the API. You can send those projects to me via zip files or send me a public link to the demo.

If you are an artist or designer, you can create assets and send them to me to be featured in demos, examples, and tutorials.

Also, a huge help for future users is writing devlogs about your adventures with the Latios Framework. You can even showcase your progress in the Discord! We love it!

If there is a particular feature that you want, build a sample project where the feature could easily be leveraged and share it with me. This gives me a concrete use case to develop the new feature against and ensure it works. Doing this cuts development time of the feature in half or more!

## Disposable Income Contributions

If you reached this section because you have disposable income and really, really want to put it towards the Latios Framework, there are primarily two ways you can help. Both involve commissioning other individuals of your choice to contribute.

In the future, I hope to have a community fund for this purpose. But currently doing so on my own would be a massive time investment. If you would like to be involved in such an endeavor, whether as a donator, a manager, or a commissioned recipient, please reach out to me privately.

### Art

Most “free” art isn’t actually free. Places like Mixamo and the Unity Asset Store do not allow for redistribution of assets in source form. Unfortunately, this restriction makes these assets unusable in sample projects. Some art is licensed under Creative Commons Attribution; however, this license is a bit vague on how to properly satisfy the conditions of attribution. Some art is given the “free” label, but otherwise does not specify how the art may be used in any way.

One of the best ways to help out framework development is to provide art assets that can be used for future experiments and sample projects. You can make them yourself or commission someone else. If you’d like, you can involve me in the process to ensure the result will be usable, but this is not a requirement. CC0 or a similar unrestricted license works best.

If you reached this section because you have disposable income and really, really want to put it towards the Latios Framework, the best thing you can do is commission an artist to create assets that can be used for future experiments and sample projects (yes, this does make feature development go faster because I can better test the effectiveness of features in more realistic scenarios). There are plenty of sites out there where you can commission an artist. Pick whoever you like.

If you are an artist, pretend you were commissioned for whatever amount you like (or don’t if that messes with your mojo). Assume whatever art is contributed will be released as CC0 within a year of its creation.

While most art will be useful in some capacity at some point (use common sense), here’s a living list of ideas that will be especially helpful:

-   3D characters
    -   While there are good options for rigid low-poly characters, characters that can make use of physics-based secondary effects such as cloth, hair, soft-bodies, or soft IK are very scarce. Characters are an important part of the Latios Framework. But it can only be as good as the art it is built to handle.
-   Environments
    -   Test environments tend to be simple and clean, while real game environments are often much messier, and will stress physics and rendering systems much harder. Environments that push the limits of the tech are awesome! But environments that can be dropped in for sample projects are also a great way to get more sample projects out in the wild.
-   Weapons and Effects
    -   It is not always obvious how all the different features of the Latios Framework can be tied together. Having ready-to-use weapons and effects can be the launchpad of better samples to demonstrate multiple features working in tandem.
-   Terrain
    -   If you want terrain support with trees and details and all that to work in ECS, then make the scene you want to see function in ECS and send it our way.

### Code

I tend to focus on code that plays to my strengths as a developer, which usually involves performance-sensitive code and complex data. Consequently, I tend to not spend time on things to improve the editor experience or reduce boilerplate. If you are looking to spend money to get someone to help me, these are the areas that someone else can easily provide a lot of value to:

-   Editor scripting and tools
    -   Most of my authoring components use default inspectors and have very limited or broken debug tooling. This can be drastically improved.
-   Source Generators
    -   I have plenty of ideas for source generators that can be used to reduce boilerplate, but often times, I struggle to justify the time investment. If you like the idea of reduced boilerplate, feel free to reach out, and I can walk you through some of my ideas.
-   Physics
    -   If you are a physics nerd, Psyshock could use your help! I’d love Psyshock to grow to support multiple different solvers and constraints and techniques that users can mix and match to craft their perfect physics engine.

If you have further inquiries, PM me on the Unity forums or DM me on Discord.
