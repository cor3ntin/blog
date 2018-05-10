---
title: "On the state of cross-compilation in the C++ World"
date: 2018-02-02
---

I wrote a series of article where I compile simple Windows and OSX applications from Linux.

I hope you enjoyed it. For me, it certainly was quite the journey. Or the beginning of one. There is lot of rooms for improvement and we left some area unexplored, including some other major Operating systems like Android and iOS. I also did not talk about debugging.

The open source community is amazing. We should not take projects like llvm, wine, darling, or even osxcross for granted.

And it actually works. We were able to build and even run applications for Windows and Mac, and it’s great.

I of course, didn’t invent anything. Boris Kolpackov demonstrated cl.exe running on Linux in 2015. But it’s not something you will find a lot of documentation for. Linux people have been cross compiling for other architectures since the dawn of times, and mingw64 is quite popular.

{{< youtube PxFrhYAYF3M >}}

There are however issues. Most issues can be sum-up as the target OS vendors making cross compilation difficult. Not compilers people, they are amazing. Packagers. All issues we had with windows, OSX and even Qt came done to legal considerations and packaging issues. Assumptions turned into black magic. Black Magic is not, as it turn out cross platform.

Don’t assume my filesystem layout, my tools or my host architecture.

If we had better, legal way to install the OSX and Windows SDK & toolchains, I expect more people would be invested in improving the cross compilation. It wouldn’t take much.

## Why Even Bother ?

Alien toolchains may seem like a novelty thing to puzzle your friend, but they are incredibly useful.

### Automation

Doing Continuous Integration on Windows and OSX is not trivial. Linux has containerization support, a great virtualization story, a huge range of administration and monitoring tools. So if you have a choice, you will probably want to run your CI server and agents on linux. The ability to cross compile to Windows and Mac makes system administration much easier. And you would need less hardware, reducing costs and idle time of each node.

### Don’t break the build

While you should have a gate keeper that checks every commit before it gets merge, if you have tested your code on more platforms, you will have the assurance that your code compile on all platforms.

### Faster development

If you write cross-platform applications, chances are, sooner or later, you will have to use platform specific APIs. Switching platforms to implement a feature takes an incredible amount of your time. The more you remain on a single development environment, the more productive you will be.

### Portability and Reliability

The more compilers and toolchains in your workflow, the more robust your code will be, and the more portable it will be. Different compilers have different warning sets and different behaviors. You can either let that fact of life be your downfall or use that as an asset!

## Testing

While wine and darling are great, they are of course not suitable for Quality Assurance testing.

If your software is supposed to run on windows 7 to 10, then you need to test it on all these platforms.

However, if you are not shipping a Linux version of your application, which you should, maybe you can run in wine from time to time.

But most your unit tests should work on wine / darling, use that to your advantage

## An Aside On Build Systems

In this series, I used QBS. It’s a weird choice. Why write about something that virtually nobody uses ? Why not CMake ?

Well, first, QBS is language agnostic. Everything pertaining to C++ and C++ tool-chains is scripted in JavaScript files. It make it quite hackable.

And because of that it lends it self to cross toolchains more willingly than other build systems. It can also build multiple targets for various target system during the same invocation in parallel. I am aware of very few build systems that have that ability. build2 is probably your only other option. and build2 is, by all accounts, and amazing build system, in some regards better than QBS ( for one, it supports c++ modules).

It has, however no support for Qt, and while this could in theory be fixed, qbs still has a major advantage.

Both build2 and QBS having full control of the build graph they can do pretty wild things. Like build in parallel multiple independent targets, for different architectures.

In fact, it can build for all the profiles it knows, at the same times. Here I build a hello world for all my compilers

{{< figure src="qbs.png" title="Your move CMake" >}}

QBS as another thing going for it. A sane, understood syntax: QML. And I truly do believe it offers the best language of any available build tools currently.

QML is about 10 years old and are clear, established rules. And of course, it would be fashionable to hate on its use of JavaScript, however, the scripting is powerful, but not too powerful as to make your build files un-maintainable. For all intent and purposes, the syntax is declarative with an intuitive logic. in that, it avoids the issues scons, waf and other suffer from.

It focus on user-friendliness rather than terseness.

And most importantly, it’s a build system, not a build system generator. It properly detects changes in files, flags, etc and rebuild the graph accordingly.

Of course build2 also has that capacity, as well as others.

### CMake however, has none on these properties.

I don’t believe being ubiquitous is a sufficient quality to redeem CMake which, in the end, is just another Makefiles generator with terrible syntax and worse documentation.

In your future projects, consider using qbs or build2 . If you are a library writer, you can offer a cmake files so that your users can link to your libraries. It would be far easier to offer a great c++ package manager if the world settled on good build systems.

QBS links to some Qt shared libraries. Of course that’s a downside. I hope it will get rewritten. Lend a hand if you can. But it’s not something that should stop you from considering QBS when starting your next project.

## What could be down to improve the cross compilation story ?

### Ask Microsoft and Apple to offer a simpler way to get a System SDK

If both Microsoft and Apple were to ship their SDK as a tarball, without restrictions on how it can be redistributed, it would be far easier to the open source community to use them and great build Apps with them. It’s a win win. Since both XCode and Windows SDK has no license costs and it’s already possible to share them, albeit not legally, it should be a win-win situation.

### Don’t bake in assumptions about hosts and target system in your build tools

Ideally, all the facilities offered by a build tool should be cross platform such that adding targets is easy. However that’s often not the case as we have seen with Qt build scripts, and the inability of QBS to deal with .plist on Linux.

Another issue that I did not touch is code signing. it’s possible to sign windows applications from Linux, the same can not be said for OSX applications. [Some open sources projects solve that](https://github.com/appknox/isign).

### Support Wine and Darling

Wine and Darling are both fantastic open source projects. Their task is however enormous. Sure, wine is great for games but they should be seen as amazing development tools.

Imagine having the iOS simulator running on Linux ?

For that, they probably need funding, company backing and developer time.

### Be grateful for LLVM

Most of what was presented here would not have been possible without LLVM. Fortunately, it’s a well funded project, but I’m sure they could use some help. Adding support for .tbd files on lld would be a cool project. I like the idea of tdb files, maybe they should be usable on all platforms ?

## Universal Toolchain Descriptor ?

I though I was being clever and original but apparently the idea was already discussed on the Cpp Slack.

A toolchain is something relatively simple and well understood, as we saw in this series. It’s a compiler, a linker, some other tools for maybe compiling assembly, stripping symbols. It’s a bunch of include paths and library paths, in rare cases a bunch of flags.

So, what if we created a file to describe any c++ toolchain, including aliens one. It would be very similar to our QBS profiles, but with a same syntax, like YAML.

We could specify a standard location for that file on a variety of systems. And build systems could read it to discovers toolchains ( instead or in addition to relying on black magic ).

Of course, it would only really work if all build systems are willing to use that file.

Do you think that is something worth pursuing ?
