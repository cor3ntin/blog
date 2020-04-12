---
title: "Build C++ from source: Part 1/N - Improving compile times"
date: 2020-03-23T16:00:00+01:00
draft: false
---

This is both a follow-up to my [CppCon talk](https://www.youtube.com/watch?v=k3Q-fPBe9Z0) and the ongoing ABI saga, which I do not expect to end any time soon.

I hope this article to be the first in a series I hope to write over the next few months.

A quick reminder, ABI is akin to a binary protocol and decides how your types are laid out in memory, how functions are mangled and are called.
As such, many changes in the source of your program that are not visible at compile time manifest themselves at link time
or runtime.
ABI stability is only relevant in the case when you try to link or load libraries that were not built in a consistent environment.
We will come back to what a consistent environment is.

Following the WG21 Prague meeting, many people are trying to solve the ABI problem. Ultimately, all the solutions being proposed boil down to:

* Partial silent ABI breaks (like was done in C++11 for `std::string`)
* Duplication of types, whether through a new name or a new namespace, or some other mechanism that ultimately boils done to that.
* Adding a level of indirection, whether COM-like, PIMPL-like, or some [semantically equivalent fancy solution](https://github.com/apple/swift/blob/master/docs/ABIStabilityManifesto.md#value-witness-table).

I do not think any of these solutions work in the context of the C++ standard library.
I might try to explain why at some point.

But, with the assumption that ABI stability has costs that go against the essence of C++, whatever solution remains, however difficult, must be the path forward.
Let's talk about building from source.

It is important to understand why people do not build from source or think it is not realistic to ever do so.
I believe the reasons are:

* Compiling from source takes time
* Big binaries take disk and memory space
* Compiling from source is hard because of build system complexity.
* Use of commercial software without access to the sources for legal reasons
* Use of libraries whose sources have been lost to time
* Use of compiled system provided libraries
* Use of some kind of plugin system

(Please let me know if I forget something that does not fit in one of these categories)

I am hoping to release a series of articles on each of these problems in the coming weeks.
**Today, I will focus on presenting various solutions that can improve c++ build times**.

## Make C++ faster to build

C++ is arguably a bit slow to compile. Slow enough that people rather download pre-built binaries
not to pay the cost of that compilation.

There are a few solutions to make C++ faster to build.

### Use better hardware

**This part is mostly targeted at employers. Especially, if you are a hobbyist, a student etc, do not think you need unaffordable hardware. Your hardware is fine**.

Compiling C++ is slow. By design. The idea is to do more at compile-time and less at runtime. Someone has to pay the cost of the abstractions and C++ chooses to make developers pay that cost, rather than users. Which is more than sensible as programs are usually run on many more machines than they are compiled on.

That doesn't mean that compilers aren't well optimized, they are.
But there are limitations to how fast a C++ compiler can be.

Fortunately, while compilers seem to not use more than one thread, it is easy to compile many C++ source files at once. C++ compilation will scale pretty much linearly with the number of parallel jobs if your hardware allows it.
The good news is that CPUs with many cores are getting cheaper to buy and to own.
With 12, 16 cores, you get to compile 24, 32 translations units at the same time. Making it easy to compile the entire LLVM toolchain in less than 10 minutes.
Of course, these CPUs are not cheap, but they are definitively a lot cheaper than a few years ago and it is likely the trend will continue.

I don't want to tell anyone that they need an expensive CPUs, or that an expensive CPUs is necessary to work with C++, but I think it's a business
decision to consider. Investing in hardware definitively made me more productive.
People in the VFX or CAD industries understand that they need expensive
hardware to be productive in their job (which doesn't preclude the nest of modest hardware for non-professional uses).

#### Hardware you say?

Here are a few things to consider:

* AMD CPUs are currently cheaper than intel'ss but might not work with tools like `rr` and `vtune`. I went with a Ryzen 3xxx and it works great for me.
* You probably want to have 1-2GB per logical core. If you go for a 16 cores CPU, you might want 48-64GB of RAM
* A fast drive makes some operation faster, notably profiling and debugging but does not seem to impact compilation a lot
* Working with C++ uses resources independently of compilation: Debugging, profiling and code indexing (aka using an ide) are taxing on CPU, memory and drives alike.

#### What about CIs?

If you administrate your Contiguous Integration platform, the same hardware recommendations apply, depending on the rate of commits and merges.

Unfortunately, most cloud services offer modest hardware, usually a couple of cores, which makes compiling large frameworks very slow, if not impossible.
It is very hard to complain given the tremendous benefits these cloud CIs offer to open-source communities for free.
But these offers are a poor fit for many open-source C++ projects.
We need more cores!
I wonder if offering faster builds while limiting the number of builds over
a given period could make sense for both users and CIs providers.

### Caching

Being able to recompile LLVM, Qt or boost in a few minutes,
doesn't mean we can afford or want to do that each time we start a build.
Which is why we talk of building **as-if** from source.

This has a very specific meaning: The existence of any caching mechanism must not be observable by the final program.
Compilation of C++ programs can be impacted by compilers, compiler flags and which headers are included, and which library are linked. In many cases, headers are installed system-wide. Hopefully only system libraries headers, We will come to that in future articles.

As such, the use of precompiled libraries (static or dynamic) or the use of precompiled modules do not constitute caching and are a good reason why your programs are ill-formed no diagnostic required.
**Do not redistribute your modules**.

Many things can be cached:

#### Tracking change

The job of a build system is to keep track of the minimal and sufficient
amount of work needed to perform a build whenever files, compilers options, and other dependency are changed.

#### Caching translation units

Tools like `ccache` and `sccache` allow you not to rebuild an object file when
unnecessary. A good build system should alleviate the need for such a tool
in many cases, but in practice, they prove very useful.

#### Modules

I do not think we can meaningfully improve the state of the ecosystem without modules.
As far as compilation speed is concerned, modules can act somewhat as precompiled headers (modules have other benefits beyond compilation speed), in that module interface need to be parsed only once.
And templates **used** in that module can also be instantiated there.
This can speed up compilation a lot, but it might be a while before we
can observe the real-life impact of modules in open-source projects.

#### Precompiled headers

Waiting for modules, you can probably benefit from precompiled headers.
CMake 3.16 has support for them, and it can be quite beneficial for third party libraries, or even for your own code to speed up to full builds.

#### Caching template instantiations

One of the most expensive things C++ compilers do is template instantiations.

A now mostly dead project [zapcc](https://github.com/yrnkrn/zapcc) aimed
at caching template instantiations across translation units, which
was benchmarked to have a 2-5x compile-time speed increase.
Unfortunately, `zappcc` is based on an old fork of clang and I do not think any work to integrate this technology to mainstream compilers is planned.

### Measure

Both [MSVC](https://devblogs.microsoft.com/cppblog/introducing-c-build-insights/)
and [Clang](https://github.com/aras-p/ClangBuildAnalyzer) have tools to profile
the most costly parts of your build.
I highly recommend using these tools if you want to optimize your build times, because
like as all performance tuning work, you will most certainly discover your intuitions
about what causes bottleneck will probably prove wrong.

### Other tricks to make your compilation faster

{{< youtube X4pyOtawqjg >}}

This video offers many insights into improving compilation speed.
Two easy recommendations are to use `lld` instead of ld and `ninja`
instead of `make`.
If you are on windows, make sure your antivirus will not inspect
every file you read and write each time you invoke `cl.exe`.

## Compilation at scale

For large organizations other tools are available to improve compile times:

* Distributed builds such as `distcc` and `icecream` can be set up to share the workloads over many machines.
* Some companies have their employees compile on powerful remote hardware through ssh. This has the benefits of giving users access to very powerful hardware while ensuring optimal resource utilization.
* Both solutions can be combined.

## Do not design for compile time

At the interface level, compile-time should be at the very bottom of your priorities and goals.
There is no secret here. Improving compile time is easy: Remove abstraction, remove genericity.
Is it what you want?
Probably not.
In this article I presented many solutions to improve compile times, you should exhaust all of them before considering compromising your interface in any way (safety, usability, performance)
for the sake of compilation times.
Some improvements can be gained by not including the headers you do not need, unfortunately, these are hard to tracks.
Victor Zverovich wrote a nice article about the [compile time performance of {fmt}](https://www.zverovich.net/2017/12/09/improving-compile-times.html) a few years ago.


## Compile times and ABI breaking standard changes

If you don't invest in your build systyem and infrastructure, maybe your build takes hours.
Days. Weeks?
But, the standard would at most make potentially ABI breaking changes once every 3 years.
That is not a lot, compared to the program not running optimally the rest of the time.
So, I dont subscribe to the idea that compile times are a reason to keep an ABI stable over
lengthy periods of time.

Of course there are many other things to consider, but that's for another time!

Until then,

Stay safe!