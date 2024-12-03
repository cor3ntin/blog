---
title: "Legacy Safety: The Wrocław C++ Meeting"
date: 2024-11-29T09:14:07+01:00
---

{{< figure src="1.jpg">}}

Someone was giving away stickers reading "Somebody Should Do Something" at the WG21 C++ Standardization meeting held in Wrocław last week, and it makes for a pretty good tagline for that meeting.

We are one meeting away from calling C++26 feature complete.
The draft will be sent to National Bodies this summer. After that, we will have a couple of meetings to fix bugs and then wait for ISO to publish the official standard, which might take a year or two.

Hard-pressed by self-imposed deadlines, I would not say we are rushing design reviews, but boy, does it seem like it sometimes!

These deadlines are somewhat ridiculous, as very few people need to care about ISO publications, and to some extent, the version of C++ you use is irrelevant; features are implemented out-of-order and often deployed in older language modes when possible.
Features test macros are good; use them.
On the other hand, we still don't have a better way to stabilize features, so publishing
the draft of the international standard is a somewhat okay cut-off point (even though WG21 will apply breaking changes well into 2025 if it feels like it). Either way, I don't think we are justified in designing features based on deadlines
(`basic_fixed_string`, `define_aggregate` come to mind).
All major features, especially reflection, have rough edges, and their design is still in flux.
However, this end-of-the-cycle rush is on par for the course, and most features should stabilize
over the next year.

I know it is sometimes difficult to form a fair view of the outcomes when reading about WG21 proceedings, so before going further, we should acknowledge that there is a lot to love about C++26.
[Reflection](https://wg21.link/P2996) and [Contracts](https://wg21.link/P2900) are poised to land in C++26, and each proposal represents
years of hard work. We also just approved [`std::simd`](https://wg21.link/P1928), a portable way to write SIMD code.
I remember the author telling me about this work at my first meeting back in 2018.
I think [sender/receivers](https://wg21.link/P2300), approved earlier this year, is the right framework
to allow C++ to thrive in the current reality of highly heterogeneous and parallel hardware.

C++26 will be a significant, exciting release.

On the flip side, I have some concerns that the committee doesn't consider enough implementer bandwidth,
especially on the library side.
Don't be surprised if some of these proposals take years to implement.
It's a somewhat tricky position to take, having a (totally unfounded!)
reputation for writing a lot of papers.
Yet, I think it's fairly important to consider the economic impact of our work,
and I wish we had that conversation more often.

And with that in mind, I guess we need to talk about safety again.
Some great work happened on that front. A whole lot of nothing, too.

## Safety, why do we care?

You'd think our newfound proclaimed interest in language safety would be motivated by our desire to improve our users' productivity or because we genuinely admit Rust has some good ideas worth
exploring. And mostly because we understand the world is changing, and our perception of code quality and sound engineering is shifting for the better. And to some extent, all of that is true.

But, seemingly, it isn't what drives the discussion.

Consider bound checking on `vector::operator[]`.
We had the technology to solve that problem in 1984. We did not.
Consider destructive moves. We had a window opportunity in the C++11 time frame. We choose not to take it.
What changed?

People at [Washington](https://www.whitehouse.gov/oncd/briefing-room/2024/02/26/press-release-technical-report/) (and [Bruxells](https://digital-strategy.ec.europa.eu/en/library/cyber-resilience-act)) realized poorly written software was a threat to national security and critical infrastructures and that they should do something about it.

Memory unsafety is widely understood to be the [most significant cause of vulnerabilities](https://www.memorysafety.org/docs/memory-safety/),
consequently, C++ (and C, obviously) found itself on the [naughty list of Big Brother](https://www.theregister.com/2024/11/08/the_us_government_wants_developers/).
[That ruffled some feathers](https://downloads.regulations.gov/ONCD-2023-0002-0020/attachment_1.pdf)!

So, the C++ leadership is reacting, with haste, to a perceived threat.
We care about safety out of fear. Is that conducive to good design outcomes?

## Reframing (language) safety.

We would like to make it close to impossible to compromise software so that we don't endanger
the lives or privacy of anyone. Imagine making Software Engineering a bit more like Engineering.
Crazy, right?

This is hard, but frankly, you can do that in C++, or assembly, or brainfuck.
It's a factor of cost.
C++ can be perfectly usable in an air-gapped system buried under kilometers of granite, otherwise sandboxed (which is how Chrome works), or by applying sufficient fuzzing and qualification testing, but that takes time and money.

Software vulnerabilities are ultimately a failure of process rather than a failure of technology.
The good people at [Crowdstrike simply had inadequate testing procedures](https://www.crowdstrike.com/wp-content/uploads/2024/08/Channel-File-291-Incident-Root-Cause-Analysis-08.06.2024.pdf), and what happened in the guts
of the billions of affected CPUs, while interesting, is not the material cause.
No tool is going to help in an environment without a strong safety culture.

Our concern is that, the more language safety becomes a prevalant goal, the less C++ offers a compelling value-proposition.

The sooner unsafeties are found in the development cycle, the less they cost, even if that means that writing code
becomes more expensive (i.e., [shifting left](https://en.wikipedia.org/wiki/Shift-left_testing)).
Some issues in C++ can only be found by running the code in sanitizers and [fuzzers](https://security.googleblog.com/2022/09/fuzzing-beyond-memory-corruption.html), which is expensive.
Writing code is a one-time cost; fuzzing must happen continuously in the product's lifetime.

Under increasing regulator scrutiny, the cost of documentation and certification increases too, such that at some point bidding on a contract with a C++ solution will not be competitive (especially as C++ has always been a costly hammer to begin with).

That perspective on language safety is essential to keep in mind.

The point is your code is not more unsafe than it was two years ago,
but interest rates on technical debts are going to keep increasing.

Somebody should do something!

{{< figure src="2.jpg">}}


## Memory Safety from first principles.

In that context, Sean Baxter presented ["Safe C++"](https://safecpp.org/draft.html).
The general idea is to introduce owning references and borrow checking to C++.

It's not just that we know this solution works; it's one of the only solutions we know to be mathematically sound.
Well, not _quite_. Whole program analysis (haha) or a language with only value semantics could work.
But, in the context of C++ (unless we are willing to impact runtime performance - Memory tagging or CHERI are still options, for example),
borrow checking is the least impractical compile-time solution to memory safety.

There is even research for [borrow checking in C](https://dl.acm.org/doi/pdf/10.1145/3652032.3657579)
and [D](https://dlang.org/blog/2019/07/15/ownership-and-borrowing-in-d/).

That doesn't mean it is _practical_. It would require a very ambitious and costly rewrite of all
code that would want to be memory-safe, including the standard library.
The deployment story is sort of awful.
But the model works, and if there is a desire to make C++ memory safe,
that would be a good way to go about it (whether what Sean Baxter proposes, or something isomorphic to it, after figuring out a gazillion details).

[Google argues](https://security.googleblog.com/2024/09/eliminating-memory-safety-vulnerabilities-Android.html)
that making small pockets of a project memory safe does reduce the vulnerability
of the overall system and has outsized benefits - I found that surprising (and a good reminder not to operate on gut feeling) -
which is an argument in favor of going in that direction.

Sean Baxter seems to believe that you can arbitrarily modify C++ and still call it C++, but we should be more
explicit about the fact that Safe C++ is more of a successor language than an extension to C++.
In particular, the incompatibility with the standard library might very well be a deal breaker unless it can be addressed somehow.
Can we make the existing STL compatible with borrow checking yet allow its use in existing code?
Can we provide wrappers that still allow cheap conversions to/from existing code?
Can we make tuple work? Are large components such as `<ranges>`, `sender/receivers`, `<algorithms>` amendable to borrow checking and relocation?
Can we keep parts of the existing library as-is and use it from safe code, under the assumption that we understand the behavior of the STL,
and implementers can qualify its behavior?
Quid of ABI?

I don't think there is room for a new standard library. We can barely afford the one we have already,
we don't have a good model to avoid ambiguities, conversions, and ADL terribleness.
Having to retrain people to that extent is unlikely to motivate anyone, either.
Maybe more importantly, WG21 works well under constraints, but give us a blank page and the opportunity to fix our past mistakes, and we will implode (and make the same mistakes, plus some new ones, too).

It doesn't mean we should not explore our options.
A successor language that would retain most of the syntax of C++, its meta-programming facilities,
powerful overload resolution, constant evaluation, etc, would be more compelling than most.

There is a lot of support and excitement for Sean's work, but it is large, scary, and ambitious,
WG21 has a limited appetite for large, scary, and ambitious - and arguably a poor track record of execution on large features.

Even setting the standard library aside, that concern is not without merit; in particular, this would be long-term work, and at the moment, no one seems willing to bankroll or spearhead it.
We need to accept that investments in C++ front ends are limited and maybe even waning.
Google and other major players have reduced their involvement in C++ standardization
(both for technical and non technical reasons), and this trend is not likely to be reversed.
Sean Baxter has shown no interest in further involvement in WG21, and frankly, who can blame him?
Large proposals also have a history of failing or crumbling under the weight of compromises.

But a lot of the criticism isn't focused on the viability of this direction.
Rather, people keep repeating that Rust has vulnerabilities too, and therefore memory safety is insufficient
and therefore, unecessary. The C++ community understands the benefits of ressource safety, constness (sorta),
access modifiers ([kinda](https://wg21.link/P2996)) and type safety (ideas that, to this day, are often dismissed by a lot of C programmers),
yet we feel the urge to dismiss the usefullness of lifetime safety. Why are programming communities so siloed?

The self-proclaimed C++ leadership[^1], in particular, seems terrified of that direction, although it's rather unclear why.
Their solution, which after the Poland meeting has stronger consensus in the Safety Study Group is named "Profiles".

## Expecting a different result

What are [Profiles](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p3081r0.pdf)?

A set of pick-and-choose-your-own-safety rules that disable some constructs such as `reinterpret_cast`,
or inject runtime checks for overflow, bound checking, and so forth.

One such profile attempts to provide a mechanism for memory and lifetime safety without modifying
existing code (peppering the code with verbose attributes somehow does not count).

Profiles, therefore, are a set of sanitizers and static analysis checks.
Their exact nature is very unclear at this point because this work is extremely light
on details.

There is some hubris in thinking that WG21 could deliver memory safety through static analysis.
People have been doing static analysis for decades, so we have a very good understanding
of the limitations. In particular, we know [we cannot achieve guaranteed memory safety without annotating most references](https://www.circle-lang.org/draft-profiles.html) - and that would still be insufficient.
(Also, if it really needs to be said, lifetime annotations must be part of the ABI to offer safety over time.)

The proponent of profiles points out that their goal is an overall reduction in vulnerabilities,
and they are not aiming for perfection. This is extremely fair. Any mitigation is worth considering.
However, we should acknowledge that there is a giant gap between 99% and 100% memory safety.
It would be reasonable to treat a fragment of code that may have false negatives as being completely memory-unsafe, so if profiles could help with memory safety, they would not help with confidence. (`-Wlifetime` has yet to demonstrate effectiveness on non-trivial examples,
so the rate of false negatives is much higher; can it even be 50% effective?).
And having a full understanding of object lifetime thorough the code would also be an [exciting opportinity for tooling](https://github.com/willcrichton/flowistry).

It doesn't mean static analysis isn't useful or that tools with a high number of false negatives don't help — in fact, I think `-Wlifetime` is pretty cool — but if I had concerns over the long-term viability of C++, that work would do very little to alleviate them.
It is unclear whether this work would adequately satisfy the requirements of governmental institutions, and that certainly would be useful information.

Implementers and tool vendors are not actively involved in the work on Profiles,
and the work on `-Wlifetime` - initially presented circa 2018 -
never reached a complete, deployable state. If there was any confidence in the outcome, surely more investments would have flown in by now.

There are a lot of concrete issues with some of the proposed profiles that would make them unfit as compiler warnings. Namely, some of the suggested rules are just stylistic, which would make them too verbose
(and users _hate_ chatty warnings; their tolerance for false positives is _0_ - lest the warning gets disabled).
The categorization by kind (such as "lifetime" and "arithmetic") would make it very hard to add new rules,
as users generally prefer warnings to be grouped by cost (compile time, runtime) similarly to what libc++ is doing,
and/or by severity and reliability. We are constantly reminded of the billions of lines of C++ code out there.
Yet, some profiles propose silent semantic changes to existing code, which we know will explode at scale.
In that context, we should be very wary of solutions that work perfectly 95% of the time.
Having code behave differently under different profile configurations also seems to me like a recipe for disaster.

Profiles have a very explicit goal of shipping in C++26 (that is, in less than 3 months).
It's very unclear to me why some safety guidelines with 0 impact on the language whatsoever need to
be the standard and not in a separate document (especially as most C++ deployments are still on C++17).

You'd expect them to be implemented, researched, or motivated, but they appear to be none of these things,
and the myriads of papers on the subject seem to recommend WG21 throw spaghetti at the wall and see if anything sticks.
I might be judging profiles unfairly, but it is difficult to take at face value a body of work
that does not acknowledge the state of the art and makes no effort to justify its perceived viability or quote its sources.

To me, this makes little sense... until you realize profiles are very easy to sell.
They reassure people who don't care about safety that they don't have to, and they reassure everyone else that there is a path forward.
And we can blame users or implementers when it ultimately fails to have a meaningful impact.

The safety group saw a paper saying "`constexpr` has no UB", let's do the same thing at runtime" without explaining how such a feat would be achieved. This is as useful as asking for a pony.
Of course, the Safety Study group unanimously voted to have a pony.
This is compounded by the fact that there are a limited number of safety and tooling experts present in these discussions.
That same paper suggests that cases of explicit undefined behaviors should be cataloged.
Which is great, and in fact, people have been doing [exactly that](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1705r1.html).

WG21 seems keen to show they are doing something, now!

{{< figure src="3.jpg">}}


## Somebodies are doing something!

The thing is, there is a lot of high-impact work happening to improve the language safety of C++,
it is just not committee-led.

Companies like Google, Microsoft, Apple, Adobe, etc all have large amounts of C++ code,
so they are actively trying to improve language safety for their own interests
(both to reduce maintenance costs and minimize attack vectors).
They just do it mostly outside of the committee.

Compiler and tools vendors are also hard at work exploring solutions to improve language safety through extensions,
warnings and sanitizers.

Emphasis on "exploring". There are many good ideas out there, but most of them aren't ready for standardization.

For example, Clang gained a flag to enforce bound safety for C arrays and pointers [-fbounds-safety](https://clang.llvm.org/docs/BoundsSafety.html).
This does require annotations, and in C++, the generalized use of `std::span` would be a better solution.
Clang provides [-Wunsafe-buffer-usage](https://clang.llvm.org/docs/SafeBuffers.html) for that purpose.

Both `libc++` and `libstdc++` introduced various modes [enforcing preconditions on standard functions](https://www.youtube.com/watch?v=t7EJTO0-reg) and some of that has been deployed internally
at [Google with great effect](https://security.googleblog.com/2024/11/retrofitting-spatial-safety-to-hundreds.html).
Google's blog post introduces concrete data on performance impact and outcome (number of bugs found and segfault rate), all of which are very
compelling.

Some projects, such as [SafeStack](https://clang.llvm.org/docs/SafeStack.html) and [Type isolation](https://security.apple.com/blog/towards-the-next-generation-of-xnu-memory-safety/), aim not to improve spatial safety but to limit the potential impact of vulnerability exploitations.
They may not be as glamorous as borrow checking, but they are effective, concrete, cheap solutions.

EWG saw and approved [a paper to enable type isolation](https://wg21.link/p2719r1) by adding `new` overloads.
That is an excellent example of actionable improvement in the standard.

There is an ever-growing list of sanitizers, including a type-sanitizer being worked on for Clang.

There is a culture shift in the wider community towards increasing language safety, which in turn means that compilers are more willing to warn on potentially dangerous code.

`-Wdangling`, `-Wdangling-field`, `[[clang::lifetimebound]]` and the new `[[clang::lifetime_capture_by]]` attributes allow to express lifetime relationship and avoid some common dangling issues.

To a large degree, the work on [Clang IR](https://llvm.github.io/clangir/) is motivated by the prospect of better, faster,
more reliable lifetime analysis: Rust, Circle, and attempts to implement `-Wlifetime` in Clang demonstrated the limits
of static analysis and CFI at the AST level.
And while there are many more motivations behind [Clang IR](https://llvm.github.io/clangir/), putting `-Wlifetime` in a TS isn't going
to make that work go any faster. Whether it would motivate other toolchains to make similar multi-million-dollar investments is unclear.

More generally, many people are doing language safety research in C++.
While that work could be better documented and advertised, and while WG21 certainly would benefit from more exposure to these ideas,
not every shower thought needs to become a paper or a TS.
Papers such as [P1179R1](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1179r1.pdf) make no effort to paint a picture of the state
of the industry and end up being outdated, even if they had novel ideas when first introduced.

Vendors can experiment with warnings and checks much faster than WG21 can react, and it remains unclear whether WG21 should opinionate on what essentially remains quality-of-implementation warnings.

Looking at all of that work, it's clear we have yet to hone on a cohesive set of solutions.
We could imagine WG21 to become a great unifying force that will distill the state of the art into a rejuvenated memory-safe C++ language
for the next 30 years, but we are not there yet, and profiles are not that.

Companies seem quite content to figure out all that memory safety funny business without WG21.
Many big tech companies formed internal memory safety task forces that consider rewrites,
interoperability with other languages, and remediations that do not involve the C++ committee.


After all, the C++ committee always tried to pretend it could evolve the language without considering or driving the broader ecosystem,
(which explains the state of package management), and so there is no need to involve the C++ committee in memory-safety solutions
that do not impact the language.

## Safety-related standard C++ features to look forward to

I am not saying the committee should do nothing. Some fantastic work is happening.

* I already mentioned [Type-aware allocation and deallocation functions](https://wg21.link/p2719)
* [Partial program correctness](https://wg21.link/P1494) aims to reduce how much Undefined Behavior can "time travel",
 such that an overzealous optimizer would not optimize your code away in the presence of UB later in the program.
* [Contracts](https://wg21.link/p2900) offer a standard way to declare asserts and pre and post-conditions in a portable,
 constexpr-compatible way with enough flexibility to encourage their deployment. Library hardening, even as a last-defense runtime check, is poised to have a more practical impact on safety than memory safety could. And contracts make that easy!
 Contracts can also improve static analysis and spatial safety.
* C++26 will have basic [saturation arithmetic](https://wg21.link/P0543)

C++ could further consider:

* Replace and deprecate dangerous constructs
* Version modules (like Rust editions) and [remove some dangerous constructs therein](https://wg21.link/P0997R0), ideas that have been promptly rejected when they were presented.
* Provide safer numeric types for narrowing and modulo arithmetic.

A C++29 epoch/edition that would remove all dangerous type/numerical constructs and other warts certainly would
be an excellent start and a bold message, even without solving memory safety.

We certainly live in exciting times and can only hope to make the right tradeoffs.

["The two factions of C++"](https://herecomesthemoon.net/2024/11/two-factions-of-cpp/)
makes the case that there is a divide between the people who care about existing code for which profiles are more applicable,
and the people who don't and dream of borrow checking.
To some extent, this is true, but it is a lot more nuanced.

Sean has demonstrated he cares a lot about the long-term viability of some version of C++ and has no patience or interest in politics,
while profiles aim to soothe the world and the powers that be.

* Profiles are alluring but do not offer meaningful improvement to the status quo. Their big selling point is "do something today",
but there is no long-term vision of profiles being useful or sufficient in 5, 10, or 20 years.
* Sean Baxter's Safe C++ is an impressive proof of concept, but we don't know how to get there (standardization, implementations, and deployments would each be massive endeavors)
* Some people with large code bases, users, or expertise provide small, targeted, high-impact changes.
* Most people in and out of the committee don't care too much for one reason or another.
 Maybe they understand that rushing a solution isn't the right move.

The safety study group took a temperature-of-the-room poll, pitting "Safe C++" against "Profiles".
Never have I seen so much anxiety expressed at the idea of taking the poll.
After 20 minutes of arguing, we ended up with a highly unusual four-way poll.

Poll that I read as "Do we prefer a solution that's proven to work but cannot be (easily) deployed
over one that we know can be deployed but hasn't demonstrated its effectiveness?"

The poll was a bit muddied by suggesting profiles could be in C++26, and we are all suffering from deadline-driven design, but in the crowded, hot, poorly ventilated SG23 room, there was a very strong consensus to pursue profiles.

__Which should we prioritize: Profiles or Safe C++ ?__

|Profiles|Both|Neutral|SafeC++|
|-|-|-|-|
|19|11|6|9|

# The 10 years outlook

Overall, even if it tried, it's very unclear to me that WG21 can have a short-term impact on
memory safety.

WG21 should, if it wants to lead, consider the shape of C++ in 10 years.
In the short term, WG21 is well-positioned to offer targeted and high-impact language changes.

If we want C++ to be viable for new projects in the long term, some improvements to memory safety (and, more generally, language safety) will be needed.

If we admit this is not viable, unrealistic, or too ambitious technically or economically,
C++ needs to work on better interoperability with safe languages,
and will need to reconsider its design priorities and its domain of applicability.

C++ is merely a tool, so we should keep in mind that a slow, careful winding down might not be
the worst outcome. It's impossible to predict where C++ will end up,
but it's probably fair to say that there will be no market desire
for new memory-unsafe languages going forward.

Whatever the direction C++ ends up choosing, there is no easy path.
And we should certainly not make hasty decisions.
A lot of that comes back to whether most C++ code has
already been written, which is a self-fulfilling belief
(and if our unwillingness to operate ABI breaks is any indication,
we already decided that we are well past peak C++).
WG21 appear to widely underestimate the willingness of [large tech company to rewrite their code](https://security.googleblog.com/2022/12/memory-safe-languages-in-android-13.html).

For example, [P3466 - (Re)affirm design principles for future C++ evolution](https://wg21.link/P3466) does not seem
to be motivated by anything but an attempt to dissuade anyone from researching whether borrow checking could possibly be viable in C++.
This paper, intended to be a Standing Document, offers some dogmatic rules presented as pillars of C++,
rules that have been ignored as often as they are followed.
Inspired by an off-the-cuff criticism of Java, we are told that "viral annotations," such as those that would
be necessary for borrow checking, should be avoided.
Of course, in a borrow-checked language, lifetime is part of types. Are types bad?
They are undoubtedly viral. Is `const` bad? What about `noexcept`, `constexpr`, contracts, conveyor functions, and... profile annotations?
Maybe it is fitting that our principles would not be self-consistent.

It might be a bit scary, but we won't find the solution to memory safety in ["Design & Evolution of C++"](https://www.stroustrup.com/dne.html).
I know D&E is a good book because D&E says it's good, but if there was ever a time to throw the book away,
it might as well be now.
Safety was not acknowledged as a significant concern in the early 90s; threat models were ~~different~~ non-existent,
hardware was widely different, the ecosystem was different, the programming language landscape was different,
toolchains had different technical and economic constraints, and the community was different.

We should not discard anything too soon. Especially not as a matter of policy.
There is a lot of inspiration to be found in existing compiler tools, hardened production software,
existing guidelines, large-scale experimental deployments, academic research, and other languages.

To quote [Chandler Carruth's excellent blog post](https://chandlerc.blog/posts/2024/11/story-time-bounds-checking/),

> [...] I want to temper our skepticism as an industry. We should avoid letting it rise to the point that even attempting to solve temporal memory safety seems like a fool’s errand. We don’t want to discourage work on the problem, simply because this avenue of solving it remains doubtful. Even though we may never find something sufficient to make C and C++ automatically memory safe, we may find ways to make our memory safe languages more ergonomic and user-friendly. We may find better ways to interoperate between unsafe and safe code in the temporal space. Fundamentally, software must shift to memory-safe languages, even for high-performance code.

The long-term success of Standard C++ in a safety-first industry will not be challenged by Rust, Carbon, or Hylo but rather by its ability to adapt to this new reality.

I am certainly excited to see how all of this unfolds. One way or the other, software will become more robust.

In the meantime, don't worry; be unit-testing.

{{< figure src="4.jpg">}}

[^1]: Neither the members of the advisory "direction group" nor the convener can represent the position of
WG21, even tough that can be confusing given the many hats they wear and the many endavor they pursue.
We should also be clear that the work on profiles does appear to have extremely strong support in WG21
