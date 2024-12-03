---
title: "The Day The Standard Library Died"
date: 2020-02-24T15:30:00+01:00
---

{{< figure src="dagger.webp" >}}


In Prague, the C++ committee took a series of polls on whether to break ABI, and decided not to.<br>
There was no applause.<br>
But I'm not sure we fully understood what we did and the consequences it could have.

I do believe none of the consequences will be good.

## What is ABI

ABI is the shared understanding libraries have about how your program is serialized,
both in term of layout, calling convention and mangling.
It is exactly a binary protocol, despite not being versioned.
<br>Maybe this is a bit complicated, so I think it is better to list what ABI stability entails:

You won't be able to use a symbol in a new version of a compiled library if you do any of the following:

* Add data member to an existing class
* Change template arguments, transform a function template to a non-template or vice versa, or make a template variadic
* Make something inline that previously wasn't
* Adding defaulted arguments to functions
* Add virtual functions

And many more thing, but these are usually the one encountered by the committee and the ones that tend to kill proposals on the spot.
I also omitted ABI breakings operations that are also source breaks (removing or modifying functions).
But sometimes, removing functions is actually a useful non-breaking change.
<br>For example, `std::string` has a `string_view` conversion operator that I want to kill with fire,
and that might be an ABI break which is not a source break - or an almost silent one-.

## Why do we want to break ABI

There are a few Quality-of-Implementation changes that could be enabled by an ABI break

* Making associative container (much) faster
* Making `std::regex` faster (it is currently faster to launch PHP to execute a regex than it is to use `std::regex`
* Small tweaks to `string`, `vector`, and other container layouts
* Better conformance: some implementations are intentionally not conforming for the sake of stability

More importantly, there are design changes that would break ABI.
In the last few of years, The following features encountered ABI concerns.
It is not an exhaustive list.

* `scoped_lock` was added to not break ABI by modifying `lock_guard`
* `int128_t` has never been standardized because modifying `intmax_t` is an ABI break. Although if you ask me, `intmax_t` should just be deprecated.
* `unique_ptr` could fit in register  with language modifications, which would be needed to make it zero-overhead, compared to a pointer
* Many changes to `error_code` were rejected because they would break ABI
* `status_code` raised ABI concerns
* A proposal to add a filter to `recursive_directory_iterator` was rejected because it was an ABI break
* A proposal to make most of `<cstring>` `constexpr` (including `strlen`) will probably die because it would be an ABI break.
* Adding UTF-8 support to `regex` is an ABI break
* Adding support for `realloc` or returning the allocated size is an ABI break for polymorphic allocators
* Making destructors implicitly virtual in polymorphic classes
* Return type of push_back could be improved with an ABI break
* In fact, did we really need both `push_back` and `emplace_back` ?
* [Improving shared_ptr](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1116r0.pdf) would be an ABI break
* `[[no_unique_address]]` could be inferred by the compiler should we not care at all about ABI

The list goes on.
I think WG21 needs to do a better job of maintaining a list of these things.
I should take note each time someone says "ABI break" in the room I am in.

### What else might we want to change?

I don't know. And I don't know what I don't know.
If I had to guess?

* In the C++23 timeframe, modularization of the standard library will face ABI concerns,
in that all non-exported symbols will have to remain in the global module fragment not to break ABI, which kinda defeats the point of modules.
* There seems to be a lot of people who believe that cost of exceptions could be greatly reduced
as a quality of implementation matter but that might require breaking ABI.
* Further improvements of coroutines may raise ABI concerns, and coroutines can be greatly improved.
* Relocation needs explicit opt-in, in part over ABI concerns.
* [Tombstone](https://www.youtube.com/watch?v=MWBfmmg8-Yo) proposals would surely raise ABI concerns.

## ABI discussions in Prague

In Prague the ABI discussions lead a series of polls, that, unfortunately, are as revealing as tea leaves, and so depending if you are a glass half full or glass half empty kind of person, you might interpret these results differently.

The basic direction is:

* WG21 is not in favor in an ABI break in 23
* WG21 is in favor of an ABI break in a future version of C++
* WG21 will take time to consider proposals requiring an ABI break
* WG21 will not promise stability forever
* WG21 wants to keep prioritizing performance over stability.

In all these polls, there is a clear majority but no consensus.
The committee is, somewhat unsurprisingly, divided.

### Reading the tea leaves

#### C++ something something
The obvious flaw in these polls is that we haven't clarified _when_ we would want to break ABI.<br>
C++23? Nope, this is a definitive no.
<br>
C++26? Some people definitively intended to vote for that, others probably voted to break ABI in C++41 or voted to
break ABI once they are retired or otherwise do not have to deal with their current project.
No way to know. The exact poll mentioned "C++SOMETHING". How helpful.

There is no reason to believe that if the ABI can't be broken now, it can be broken later.
People who need stability lag years behind the standard by quite a bit.
So, if we don't break ABI now, people would have been relying on a never-promised ABI for over a decade, maybe two.
The simple fact that we had this conversation and voted not to break ABI tends to show that the ecosystem
is ossifying and ossifying fast.
Each passing day makes the problem a bit worse and more expensive.

I have no confidence that the poll, if taken again in 3 years would be any different.
It's like climate change, everybody agrees we should invest in that problem _someday_.
Let's ban diesel vehicles in 2070.

Anything that is not planned to happen in the next 5 years has exactly no teeth at all.

#### considering proposals breaking ABI

WG21 voted to spend more time on ABI breaking proposals.

This can mean a few things:

* We can waste more time in one of the busiest rooms of the committee leaving less time for proposals that have a better chance of moving forward, but ultimately rejecting the proposal anyway
* Finding non-breaking alternatives (more on that later)
* Operating partial ABI breaks (more on that later)

#### Prioritizing performance over ABI

This one was like asking a 5 year-old whether they'd want a candy.
So we voted to care about performance.
Although, alarmingly, many people voted against.

My interpretation is that the committee wants its cake and eat it
too. Which is not possible.

<!-- https://twitter.com/blelbach/status/1228962495865507840 -->
{{< tweet user="blelbach" id="1228962495865507840" >}}

Stability and ABI ultimately, given a large enough period, run afoul of one another.
<br>
This poll was important though: It touches a fundamental question:

**What is C++ and what is the standard library?**

The words touted around are "performance" "zero-cost abstractions" and "don't pay for what you don't use".
**ABI stability goes directly against all of that.**

## Far-reaching consequences

{{< figure src="birds.webp" >}}


I believe, quite strongly, that not breaking ABI in 23 is the worst mistake the committee ever made.
And I'm sure some people are convinced of the exact opposite.
<br>Regardless, here is what is going to happen as a result of that decision:

### Education nightmare

Let's be very clear. Programs that rely on ABI probably violates ODR somewhere, are probably
using incompatible flags that happen to work.

New programs should be built from source, we should have build tools designed around compiling
sources files rather than collections of libraries fetched from random places and hastily stitched.

Yes, building from source is something that is hard to achieve.
We should encourage a mature ecosystem and seamless compiler updates.
We should find ways for people to benefit from new compiler features in months rather than in decades.
We should encourage correct, reliable, scalable reproducible builds.
We should encourage easy to import source libraries and a thriving ecosystem of dependencies.

By not breaking ABI, the committee is clearly stating that they will support your ill-formed program forever.
No you shouldn't link against apt-installed c++ system libraries (which are intended for the system), but people will, the committee might as well have given its blessing.

It's a huge step backward.
How are we supposed to teach good practices and build system hygiene if there is no incentive to?


## Loss of interest in the standard library

The estimated performance loss due to our unwillingness to break ABI is estimated to be **5-10%**
This number will grow over time.
To put that in perspective

* If you are a [Big Tech](https://en.wikipedia.org/wiki/Big_Tech) company, you can buy a new data center or pay a team to maintain a library
* If you are an embedded developer, 5% might be the difference of your software running, or having to buy a more expensive chip, which might cost millions
* If you are a game company, it might be the difference between your game being great or your user vomiting in their VR headset
* If you are in trading, it might be the difference between a successful transaction or not.

I think in any case it's the difference between "I should use C++!" and "I should use the standard library" and
"Maybe I should not use the standard library", up to "Maybe I should not use C++? Maybe I should use .net, julia, rust?". Of course, there are many other factors in that decision, but we have seen that happening for a while.

Many game developers are notoriously skeptical of the standard library, they developed alternatives, for example, EASTL.
Facebook has folly, Google has Abseil, etc.

This can snowball.
If people don't use the standard library, they have no interest in improving it.
Performance is what keeps the standard library alive. Without performance, a lot less energy is going to be pored into it.

<!-- https://twitter.com/TitusWinters/status/1224377740306132998 -->
{{< tweet user="TitusWinters" id="1224377740306132998" >}}


## How could the committee address ABI-breaking proposals?

A few things are being proposed to ease the pain of not being able to break ABI:

### Adding new names

{{< figure src="monster.webp" >}}


This is the obvious solution
If we cannot fix `unordered_map`, maybe we can add `std::fast_map`?
There are a few reasons not to do that.
Adding types in the standard is expensive in terms of education and cognitive overhead and
the inevitable thousands of article trying to tell you which container to use.
Which of `std::scoped_lock` or `std::lock_guard` should I use? I have no idea. I have to look every time.
There is also the issue that good names are finite.
It adds runtime cost as containers have to be constantly converted from one type to the next,
it makes overload sets unmanageable, etc.

Ironically, there is a lot of overlap between people who advocate for this solutions and people who think C++
is too complicated.
Adding duplicated types do not make C++ simpler.

### Oh but we could have accepted this proposal

Some implementers are claiming that some proposals that were rejected
for being ABI breaking were in fact not, or they could hack around a non-ABI breaking solution.
A bit hard to swallow for my cynical self.
The fact is, they never proposed such solutions before and the cases in which this could be applied
are limited.
Supposedly the ABI Review Group (ARG) is supposed to help in this regard, but again they will probably
recommend using a different name.

### Partial ABI breaks

The idea would be to break ABI for specific type or function rather than to change ABI for all programs at once.
The issue is that instead of a nice link-time diagnostic, this solution tends to not manifest itself
until load time and is otherwise very surprising.
The committee tried that in C++11 by changing the layout of `std::string`, and it was bad.
So bad it is used as an argument against ever breaking ABI ever again.

### One more level of indirection

One solution to some ABI issues could be to access the data of a type trough a pointer
such that the layout of a type would only be that pointer.
This corresponds roughly to the [PIMPL idiom](https://en.wikipedia.org/wiki/Private_class_data_pattern)
which is used extensively in Qt for ABI reasons.
That would allow adding data members but would not relax the constraints around virtual members.

More critically, we are talking about adding a pointer indirection and a heap allocation to everything that might be at an ABI boundary.
In the case of the STL just about everything is designed to be at an ABI boundary as it is a collection of shared vocabulary type.

The cost of that would be huge.

There might be several proposals in that design space.
Notably, a few proposals are looking into making it a language feature.
Supposedly, you could either choose between performance or stability,

Ironically, making standard types into PIMPL types would be...an ABI break.

### Rebuilding our code once every three years

Just a thought.

## Furthermore, I think your proposal must be destroyed.

Paradoxically, C++ has never been more alive. In Prague, 250 people worked on many things,
including:

* Numerics
* Linear Algebra
* Audio
* Unicode
* Async I/O
* Graphics
* ...

All of these proposals have in common that they are necessarily more opinionated than most of what
we have in the standard today, they are trying to standardize things that are areas of active research
or in constant evolution.

In particular, many Unicode algorithms are not stable over time.

Then there is the huge ugly can of worm that is networking.
It is hugely irresponsible to put something in the standard that has security implications without having the ability to fix it.

As C++ decides to be stable, all these proposals need to be killed. With fire.
I don't want them to be killed. But they need to be.
They probably won't be.

The very best outcome is that we don't make mistakes and that we standardize the state-of-the-art in a given C++ version and then let things decay slowly, unable to fix them.
(In the case of the networking TS, we seem unwilling to change anything, so we are looking at standardizing what was
the state-of-the-art a decade ago, which we know can be improved upon dramatically. A story for another time.)

But of course, we will make many, many mistakes.

<!-- https://twitter.com/hyrumwright/status/1228833676240146434 -->
{{< tweet user=hyrumwright id=1228833676240146434 >}}

Some mistakes are made conscientiously as being the right trade-offs at the time
while others will remain hidden for years.

Time goes on but the standard library stands still.
Trades-off become regrets, and regrets turn into bottlenecks.

Many mistakes are unfixable because they are burned into the API and there is a collective understanding
that API changes simply cannot be.
But many mistakes could be fixed, would we be willing to break ABI.

C++ will still be around in 40 years.
If we fail to acknowledge that things will need to change in unpredictable ways at unpredictable times, the only winning move is to not play.

It is clear that the standard's associative container failed to be relevant for more than a decade,
why think that bigger proposals would have any more success?

Your proposal must be destroyed, my proposals must be destroyed.

## Could the committee even break ABI?

Many believe that the committee could simply not make that decision because
implementers would simply ignore the committee.
That entire thing was a bit of arm-wrestling and the committee didn't play.

The thing is though, implementers have users and users are ultimately the ones who have to realize what trade-offs are forced on them.

Many people rely on ABI by accident rather than by choice.
Many people rely on stability, because frankly, who wouldn't like to be able to?
But like everything, stability has a cost, and the entire C++ ecosystem is paying it.
