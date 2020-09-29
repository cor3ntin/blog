---
title: "What is the standard Library?"
date: 2020-09-20T10:46:30+02:00
---

<div style="text-align: center;">
<a href="https://xkcd.com/2347/">
<img src="https://imgs.xkcd.com/comics/dependency.png"/>
</a>
</div>

---
**DISCLAIMER**

The following represent my opinions, not that of the C++ committee (WG21), any of its members or any other person mentioned in this article.


---

I think the most fundamental work done by WG21 is trying to answer meta-questions about itself.
What is C++, what is its essence, what should we focus on? How to evolve a language with a growing community,
a growing committee?
A language that is deployed on billions of devices, with an estimated 50 billions actively maintained lines of code.
During a CppCon panel last week I was asked about stability VS evolution. This is a hot topic,
one that may never stop being on people's mind.

There are a lot of interesting meta-questions worth asking about the standard library too.
A question that comes over and over again, is what does it mean to deprecate something, why and when.
Another is what to put in there to begin with.

I wrote a few times on the subject [before](/posts/what_should_go_in_stl/),
hopefully, I will be self-consistent. Not promising anything!

Bryce Adelstein Lelbach, then chair of LEWGI coined the phrase
> Knowing that our resources are scarce and our time is limited, do we want to give more time to this proposal?

This has become somewhat of a meme in the committee.
Since then, we shipped C++20, Bryce became chair of LEWG (which is a super difficult job that he does brilliantly),
and oh. There is this pandemic you might have heard about.

Never have the scarcity of our resources and the limitedness of our time be more apparent.

We try to make the best of the situation, but I think it's fair to say that WG21's output is
reduced. And frankly, we cannot ask of anyone to prioritize C++ standardization with all that's going on right now.
But even at the best of times, C++ library design is a costly, lengthy process involving a lot of people.
Which is good, [Linus's law](https://en.wikipedia.org/wiki/Linus%27s_law), plays a huge role in the quality of the standard library.

I don't know that this used to be a question anyone asked. For a very long time, there were few enough proposals
that they virtually could accept all of those they liked.
At the beginning of the committee, there even was a single pipeline for both language and library.
We can argue whether the separation of rooms we have today is always sensible.
Most of the time library features can be added without new language proposal, but at the same time ADL, customization points,
overload sets etc have been growing pain point for the library, and no organization can escape Conway's law.

Anyway, the influx of new proposal has grown enough in the past few years that LEWG has now the luxury
and the burden to chose which direction to go in and how to use its far too limited time.

## Standardization is expensive

I think the life cycle of a library feature goes a bit like this:

* Someone floats an idea and write a paper
* The paper is matured over 1-10 years
* There is some latency for implementations (6 months - 5 years) - at least 3 or 4 implementations
* There is a ton of latency in deploying toolchains where people can use them (this might be a story for another day)
* There is a literal ton of people writing blog posts / textbooks / conference talks about that one feature
* Then there is a slow adoption and debate about whether adopting the feature is good or not


And every step is resources constrained. Deprecation and removal is also very slow.
People are still using components that were deprecated 10, 20 years ago.

Critical flight software at NASA is estimated at 1000$ per line.
I wouldn't be surprised if the standard library costs more.
And so, one of the way to answer
"Should that piece of code be in the standard" can maybe be reformulated as "Would this benefit from the standardization process?"

Fundamentally, that makes the standard library a bad package manager.
If the only motivation for something to be in the standard is to palliate to the lack of good package managers,
then it's probably not a good motivation.
(Especially as the standard library is super bad at availability. it will be years until `<ranges>` is everywhere.)

Of course, that argument falls flat if you consider `std::vector`. It doesn't *need* to be there, but we are all sure glad it is.
So there is an _universality_ argument to be made too.
If something is universally useful (for example, 90% of programs would use it), then it starts to be a very compelling
feature for the standard library.

Some features can't live anywhere but in the STL:

Type traits, and everything that needs or benefits from compiler magic and intrinsics.
[source_location](http://eel.is/c++draft/support.srcloc), [std::stacktrace](https://wg21.link/p0881r6), [encoding detection](https://wg21.link/p1885r2),
Reflection support library and other introspection capabilities, [std::unreachable](https://wg21.link/p0627r3), [std::assume](https://wg21.link/p1773r0),
[std::embed](https://wg21.link/p1040r6).
All of these are magic and rely on the compiler. In other words they cannot be implemented portably outside of the standard library.
These are necessary for communicating between user code and compiler, and are the basis of higher-level components.
A logger would use `std::source_location` for example.

This is especially true of reflection: until C++ gets reflection, an entire class of program cannot be written.
Pattern matching make it possible to write cleaner code. Reflection make it possible to write... code.
Code that you cannot otherwise emulate in standard C++, regardless how much you try.
And that can be pretty expensive across the industry.

So library components that make new things possible are high on the list of the things I think should go in the standard library.

Then, even more obvious, come amelioration to what's already there.
As standard types get deployed, the committee has to improve them. Both in response to usage experience and to increase synergy between types
or add obvious features that were missing, bug fix, support for new language features.
As such, this [std::string::contains](https://wg21.link/p1679r3) proposal might not be the most exciting that will land in 23,
but it might be one of the most useful for many people.

This is the rationale for my own  [thread name proposal](https://wg21.link/p2019r0).
It is not possible to name threads created by `std::thread`,
and people who rely on that (for ex. the game industry) need to write an entire thread class to replace `std::thread`, just for this extra piece of functionality.
Other people might give a name to their threads if it's easy, but might not bother if it implies reimplementing `std::thread` themselves.
The cost/benefit of using a feature decreases if that feature is present in the standard library. But that is mostly true for small quality-of-life features,
not larger features that are application critical.

There are also vocabulary types: types that are designed to be the glue between libraries, a universal language for interface boundaries.
These get used everywhere. We spend a lot of time getting them right because of this by-design pervasiveness. `std::span` might simultaneously
be the simplest type of C++20 and also the one that took the most work getting right.
One example of glaringly missing vocabulary type is [`std::expected`](https://wg21.link/p1059r0):
A type that is a bit stuck in another super important meta-conversation: What are the error handlings mechanism that should be used and promoted in C++?
We still have to answer that question.

But... I am not sure types are the right kind of entities at interfaces boundary.
Concepts make for a better vocabulary because they prescribe interfaces, not implementations. (`span` is nothing if not ""template erasure"" over any `contiguous_range`).
So maybe the standard library should mainly provide concepts to tie all 3rd parties together? The `<concept>` header is a good start here.

There is however an issue with just trying to add concepts in the standard.
Concepts are informed by concrete types and how they are used, they cannot be pulled out of thin air. Not without making mistakes anyway, and [concepts are not amendable to mistakes](https://www.youtube.com/watch?v=v_yzLe-wnfk).

So, it would seem that if the STL is to have concepts, it need algorithms.

{{< tweet 990390059579789312 >}}

Good algorithms are agnostic of domain-specific knowledge and can be used in the widest variety of situations.
Algorithms are not useful to people who write games, or people who make microcontrollers, they are useful to people who write C++.
I'd write better code if I was able to recognise more algorithms.

A focus of C++20 was concepts and ranges, and I hope this remains the case in future versions of C++. views in particular are one of these things people might not actively looked for if available by default.
I sure think `views::product` is more maintainable than nested loops, but I might not try to find a library that has it, if it's not in the standard.

So, magic types, vocabulary types, concepts and improvements of existing facilities. A good list of what might be LEWG priorities.

But what about networking, Unicode, processes, 15D graphics, audio, a web engine, ML facilities, JSON parsing, crypto, blockchains, Http, event handling, regexes that don't suck, etc?

These sure make a great front page cover.
But here is the thing:

WG21 is... kinda bad at design?
Not because we are inherently inept, but because library design is fundamentally hard.
And what we understand to be good library design is a somewhat fast-moving target.

The STL is the foundation of the entire C++ ecosystem. And often used to demonstrate how to use new features and write good code. And we try to cover as many use cases as possible.
Design is finding a path in the maze of the design space, and at each crossing, we go in the direction that we think is the most widely useful.
Problem is, the numbers of crossings grows exponentially with the number of abstraction layers or the complexity of the domain.
It soon becomes impossible to please everybody, or even a majority of people.

`vector` has a few knobs: allocation strategies, growth strategies, error handling etc. It's a manageable number of knobs that we can tweak to make most people happy.
Not everyone though. `folly::fbvector` is facebook's implementation of a vector with for example a different growth strategy and [relocation](https://wg21.link/p1144r5).

The higher level you go, the more knobs you can tweak. Design is inherently opinionated.
Everything is a trade-off. The trade-off of making too many trade-offs is sacrificing universality for higher-level features.
Opinions are by nature divisive and here I thought that the STL was supposed to be general-purpose.

At that point, the crowd usually suggests that WG21 members should step away from the whiteboard and instead standardize a widely deployed library.
A library which by essence is opinionated. But also, not necessarily coherent with the rest of the standard.

Back to the beginning: There has to be a value beside increasing availability to put something in the standard.
A high-quality open-source library that is widely deployed reached its target audience, and "standardizing it" (a process which involves trimming features and freezing it in time), represents a bad value proposition.
As perfection does not exist (only Haskell is perfect), standardizing is an opportunity to improve upon existing practices.
Existing practices which might be a few years behind the state of the art.
If the C++ committee is 10 years ahead of the bulk of the industry, and standardize practices that are 10 years behind, it means that we inflict 20 years old technologies
to C++ users, for the next 30 years.
This is, depending on how you look at it "well tested and understood", or no longer viable in the industry, because some high-level components (graphics, networking)
are fast-changing design spaces.

But also, as I said earlier, the committee might not be the best place for innovation, so it would seem we are stuck between a rock and a bad place as far
as our ability to offer useful high-level features is concerned.

The cost is the same either way: bitrot in the standard library and a waste of our time and resources, which are limited and scarce.
And also the industry-wide cost of dealing with the fallout.

This then becomes an exercise in risk management:

* We can standardize something well understood that might be considered outdated by the industry and not used (`<locale>`)
* We can try something new and see 10 years later how wrong we were.

WG21 has examples of both.

I worked for two years on not requiring `input_iterator` to be `copyable`, which might sound like a small change, but allows non-regular iterators,
modifying something that hadn't change since 1998, and was designed by Stepanov.
We added that to the standard at the very last meeting, with an implementation but no usage experience.
A huge majority of the committee agreed with me that it was a useful change (💙), but the risk was non-zero.
Did we break something? Because it changed a fundamental concept, it was now or never and we decided now was better than never.

A less personal but more canonical example was `std::from_chars`, although in this case, maybe the risk was taken out of ignorance
rather than a carefully weighted analysis. It then took about a year of work to implement and I think it was worth it,
C++ now has an efficient facility that I'm sure will be used at the bottom of everything.

On the other side, we shipped coroutines after having found some [limitations and desirable improvements](https://wg21.link/P1745) in the current design.
We were concerned about applying these changes at the last minute.
"Should we delay coroutines?" is a conversation I had many times.
Ultimately, the committee decided that the cost of depriving the community of coroutines for 3 more years was greater than the benefits of the proposed changes.
Time will tell. 3 years seem like an awefully long time now, but in 10 years it will seem pretty inconsequential.

## Everything is terrible over time

Very few software projects survive the test of time. And, all things considered, the STL which is about 30yo is doing pretty well at remaining current
and modern. A lot of it is probably due to the at-the-time forward-looking nature of iterators and their strong mathematical underpinning;
the bits that consisted of crudely shoving C and POSIX into the standard didn't fare so well.

And so, the goals of the STL are only aspirational: We may deliver what we consider interfaces and performance that are leading at some point in time,
but as years pass, features that were once state of the art become barely useable.
We are doomed to fail over time, the only question is how fast things degrade, and how can we minimize that.
This is especially important for composability:  If our understanding of how to write good C++ changing over time, old features might not play well with new ones.
Integrating ranges with old containers is an ongoing challenge.

<div style="text-align: center;">
<img src="building.jpg"/>
</div>

And sure, some of these questions are impossible to answer without a crystal ball. But do we even try?

We now put a comparative table of examples of code to show the impact of a proposal in papers, which we call Tony Table (the idea came from [Tony Van Eerd](https://www.youtube.com/watch?v=Zx_Tjp9WIII)). Maybe we should also try to describe how we expect each proposal to impact the ecosystem and standard over time. We could call that the [Titus hypothesis](https://www.youtube.com/watch?v=zW-i9eVGU_k).
I don't expect that WG21 would be thrilled by that idea, it might impede many proposals.
Polymorphic Allocators, which were added in C++17 are already unmaintainable. That tends to happen when a language tries to offer both virtual functions and ABI guarantees.

Frankly, I have no idea how my own proposals would stand the test of time. Probably quite badly.
Interestingly, I found that the subset of libraries features that are the more widely available (`constexpr`, allocation-free, exception-free) do better over time.

And because of that,

## I don't think we are ready for high-level features yet

I realize how baffling that sounds. C++ is 40 years old, surely the committee can manage more than `<vector>`?
The committee has both had recent successes and a desire to expand the standard library: `<chrono>`, `<format>` and `<ranges>`
are good example of that.

I don't think every component is a complete success either: `<regex>` has ABI baked implementations issues that make it unreasonably slow,
`<filesystem>` has an unusual error management mechanism that makes it somewhat clunky to use and some performance/security issues.
And let's not mention older large features (`<iostream>`, `<locale>`, `<random>`)! As I said, it is a learning process and we should try to understand why some bits of the standard are ultimately less useful than others.

## Implementers are spread thin

There is a surprising low number of people working on implementations.
Part of that is because working on implementations is fundamentally hard, part of it is because implementations' commitment to older standards, `_Ugly` name etc makes it less appealing for people to contribute.
And mostly because the financial investment companies make in the standard library is maybe not sufficient to match the growing rate of the standard.
Not all implementations ship C++17 yet. `<filesystem>`, `<charconv>` and parallel algorithms required significant investments.
This is leading to implementations sharing code or considering to do so.
I think all of that was weighted in the open-sourcing of Microsoft's STL.

We also need to consider that implementers, while brilliant engineers, are not perfect.
It is unlikely that an implementation will be the most efficient possible implementation of some proposal, for any given high-level feature.
People working on the STL cannot all be expected to be regex/networking/graphics experts.
`<regex>` as such might never perform as well as PCRE, on which many people have spent years, if not decade of constant optimization work.
Same for graphics, networking, audio, linear algebra or any large component.

Unfortunately, there is some disconnect between implementers and the committee here.
WG21 will often make design choices intended to maximize theoretical efficiency (at the cost of usability),
even if that efficiency will never be realized in actual implementations.

If it is critical for your application to scan directories as fast as the system will allow, `<filesytem>` might fall short.
This is true of networking especially. Whatever the network TS ends up to be, it is unlikely to be used in high-frequency trading and other applications with the same requirements. Yet WG21 operates on the basis that it might be - the expert friendliness of it all would tend to show that the non-high performances use cases were not considered.

Implementers often can not improve on bad performance because of self-inflicted ABI constraints. `<regex>` is as good as it's gonna be withing these constraints.
`std::regex` is also a good example of what happens when faced we many paths in a design space, [WG21 shies away from picking one](https://en.cppreference.com/w/cpp/regex/syntax_option_type).

And the committee doesn't always have the domain expertise either.
Being part of the SG-16 Unicode study group, I know how hard it is to build a common understanding both within a study group and with the committee as a whole.

And so, if the domain knowledge is concentrated in the mind of the author and a few people,
we come back to the question we started with: Is the value of a component in its implementation or its specification?

{{< tweet 1306113740572516352 >}}

It is definitively a difficult question to answer, what is clear however is that WG21's ability to deliver a richer STL within the bound of the current
modus operandi simply does not scale.

Large features will be added at the cost of small ones. Many people seem to want the networking TS to make its way to the standard.
Few understand that for that to happen, many more small quality-of-life papers will have to be sacrificed.

## The network TS

I talked about [the network TS before](/posts/humble_wishlist/) along many other C++23-targeting proposals, some of which are not even on anyone's radar.
The main issue with the networking proposal is that it has relatively little to do with networking, and more with async execution models.

The whole committee seems to agree that a good execution model is needed, but not about what is the best path through the design space.
So we have a few proposals currently progressing at the same time, trying to accommodate each other, sometimes.
But the network TS still doesn't see value in sender-receivers, which will make it incompatible with sender-consuming algorithms.

And this worries me because what the "executor" proposal (which should really be called executors and scheduler/sender/receiver proposal; Faced with 2 different paths we seem so far to commit to both, at the cost of increased complexity) is trying to do is to define the basics of async algorithms, the same way iterators are the basis of sequential algorithms.
This is where WG21 can really make a difference in that space: Finding the fundamentals that are agnostic of current or future systems and hardware.
We need to know what an asynchronous operation is before we can define any. Or at least we need to do both at the same time.
We also need to understand how error management should work in this context, even as we disagree on how error management should
work in synchronous code.
A concrete executor type (even something as simple as a threadpool) might simply have too many knobs for the eventually standardized solution to meet the universal usefulness criteria.

If the value of the STL is not in `std::vector`, but in the design of iterators and `<algorithm>`, same should be true of asynchronous programming:
Allowing many 3rd party libraries to compose well ought to be fundamentally more important that allowing polling non-secured sockets like it's 2005.

I think this fundamentally applies to all of the higher-level proposals that have been or will be considered:
Given that our resources are scarce, cross-domain proposals serve more users.

It doesn't mean that there should never be a `<network>` component, but I would like WG21 to expand carefully and consider
the extent to which networking, io, processes, etc share commonalities - composability is important.

And doing so, we will realize that there are a great many knobs to tweak, would the result please everyone?
Which platforms and users are we considering?

When we standardized a well-behaved horse, maybe we can look at adding a cart.
I can only hope that the fact that "there exists a proposal since a decade, and someone put a lot of effort in it" doesn't obscure the bigger picture.

## Sorry if you thought I had the answers

Many still believe the STL can't be used on micro-controllers and other constrained devices. Are they right? Should that be a priority?
Or should domain-specific features be amongst the [Goals and priorities for C++](https://wg21.link/p2137r0)?
How do we balance risks when ABI should dissuade us to take any?
What is the place of the STL in a wider ecosystem? Can the STL scale?

In the end, the STL is not different from any software project. Software takes more time to create than rot, and as time, money and people
are very much finite, every feature is added at the expense of another.
Until WG21 finds ways to reduce the overall cost of features, I think the STL should play to its strengths: cross-domains foundational work.
