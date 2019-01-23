---
title: "San Diego Committee Meeting: A Trip Report"
date: 2018-11-16T13:42:44+01:00
---

As I left [Rapperswil](/posts/rapperswil) earlier this year, I said very firmly that I would not go to the San Diego Meeting.

Crossing an ocean to work on C++ 12 hours a day for a week is indeed madness.

And so naturally, I found myself in a San Diego hotel straight from the 60s, to do some C++ for a week.
With the exception of the author of this blog, all people there are incredibly smart and energetic,
and so a lot of great work was done.

{{< figure src="04.jpg" title="San Diego" >}}

# Concepts

The adjective syntax prevailed, after a few years of struggle.
I do believe this syntax to be the best solution as it is both terse enough
and unambiguous. A few pain points in the language can be directly attributed
to ambiguous syntaxes (or rather, identical syntaxes that have different semantic meaning depending on context),
so I'm glad we choose an unsurprising solution over an overzealous terseness.
We actually reached agreement incredibly quickly, despite the adjective syntax not ever have been discussed before.
I do believe this syntax can be further improved over the next few years but it is a strong basis we can build on top of.

The terse syntax was important as it allows to nicely blur the lines between generic programming and non-generic programming,
simplifying the language which I hope will ultimately result in people getting a better intuition of C++ and generic
programming. Perhaps, more importantly, it makes constraining functions easy enough that the reign of syntactically
unconstrained functions might be over.

Finally, this was the last piece of the Concepts puzzle for C++20, so we might see GCC catch up to the standard soon.
Clang will probably follow in the next few months. I was also told Microsoft is actively working on concepts.
The point is, C++20 is closer than you think!

# Coroutines

Core Coroutine is slowly decaying into a solution ever so closer to the TS.
We decided to go forward with the TS until that decision failed to reach a strong enough consensus in plenary, for the third time in a row.

But I think most people are aware that we need a solution sooner rather than later.
Lewis Baker, the author of [cppcoro](https://github.com/lewissbaker/cppcoro), is actively working on solutions to simplifying the TS,
complexity being one of the concerns expressed.
The different solutions on the table are, however not necessarily either-or and in fact
people working on these different approaches are working with each other.
Moreover, a lot of people and large corporations who actually use coroutines as specified by the TS
find them very satisfactory for their use cases.
So, while this process might seem chaotic from the outside, there are reasons to hope that the result we get, hopefully
in 20, will be able to fully satisfy everyone!

However, we are getting dangerously close to the point it will be too late for coroutines to be merged in the WD for C++20.
I really hope a decision will be taken soon!

{{< figure src="03.jpg" title="Point of no return" >}}

# Modules

Modules are on the verge of being merged in the Working Draft. Which from a language standpoint is great.
We hope to see some implementations before Kona. The Merged proposal is, in some aspect more practical than the TS
but probably needs more time to fully bake.
I am still convinced that legacy imports are probably a long-term mistake but they might be a necessary evil.
I remain unsatisfied with the tooling story, but we hopefully have sometimes before 20. More papers to be written I suppose.


# Networking

We decided to postpone the discussion of networking to 23, a decision fully supported by major ASIO users that were in the room.
While widely used and mature, The network TS comes from a world without coroutines, lambdas, and executors and we want to make sure that
we offer the best possible asynchronous framework we can.

Not rushing Networking also give the committee the opportunity not to rush executors,
which we can not afford to not get right as they are the cornerstone of every asynchronous utility to come.

I know this decision will disappoint some people, but I'm pretty confident that it will become evident over the next decade that this
was the wise thing to do.
In the meantime, use ASIO, it works wonderfully.


# Text Processing and Unicode

Unicode met for its first official, in-person meeting.
We came up with a long-term plan, starting with encoding and someday have a replacement for std::regex fully compatible with the Unicode
standard.
It's a tall order, but there is no doubt in my mind this group will get amazing results.
We voted `char8_t`, a type suitable to represent utf-8 encoded data, in the standard.
We are also working on named character sequences for C++20.

The main theme of this meeting was how to best design a Unicode sandwich and deal with encoding at system boundary.
Part of that work will be to convince compilers and os vendors to use Unicode everywhere, even though we plan to have a good story for exotic platforms.
Exciting stuff!


# Reflection

I assisted to an SG7 meeting the reflection group, and overall, it seems like reflection will be _the_ killer feature of C++23.
I think the current question is whether implementers can give us the unicorns we want. They did not say no. They were very reluctant to say yes.
The best possible solution seems to be a constexpr, strongly typed, value-based solution.
It might look something like that:

```cpp
    constexpr std::meta::class_info classInfo = reflexpr(my_class);
    constexpr std::meta::function_info fInfo = classInfo.functions_by_name(f)[0];
```

Please don't read into the particulars. I made these names up a few thousands of meters over Texas.
The point is that the goal is to have reflection look and be regular c++, using regular containers and algorithms.
Reflection is the driving force between making as much of the language `constexpr` as possible.
Meta-programming is hard and slow, so we try to move away from it.

This is early days, a lot may happen in the 23 timeframe!

# Freestanding

We had a fun freestanding evening session. There was a lot of interest for Ben Craig's amazing work, and we tried to
define what freestanding is and ought to be.
Expect a paper about that in the post-mailing list.
The general idea though is that we want to make sure it's easy and economically viable for hardware vendors to put C++ in your toaster.

Michael Caise explained than bringing chip vendors on board will be as important as specifying clearly freestanding
in the standard and the standard library.

A major part of the discussion focused on exceptions and how the committee should consider that more than 40% of C++ developers
use `-fno-exception`

# Tooling

We want unicorns and we want them now, but it seems difficult to get unicorns.
Some companies expressed an interest in unicorns with 3 corns.
If you want to learn more about the tooling session, I invite you to read [René Rivera's trip report](https://robot-dreams.net/2018/11/14/2018-cppsan-wg21-trip-report/).

Later that week we talked about `std::compile`, and while we agree it's not the job of library evolution to concern itself with compiler flags,
we should probably try to improve on the status quo.
Several people suggested the idea of a module-level syntax to affect some compiler behavior,
for example to disable exceptions, RTTI, or even to modify the handling of floating types.
It looks like a very interesting area to explore and something that might be in scope for the Tooling Study Group!

{{< figure src="01.jpg" >}}

# My papers

The committee somehow decided to prioritize all my papers for C++20,
knowing that our time is limited and this will leave less time for other work.
What that means is that I will have a busy couple of months!

## Movability of single-pass iterators

A strong interest was expressed by LEWGI (Luigi) to see move-only iterators supported in the ranges namespace
And the idea of a tagless-iterator classification also made consensus.
There is no guarantee this will move past LEWG, but as a few people pointed out if we were to rewrite the STL today, non-forward iterators
probably would not require copyability and, in the absence of a time machine, Ranges are the next best thing.

## Ranges constructors

I will have to change the design a bit, but an easy copy of containers of different types and view materialization has enough interest that I'm hopeful a satisfactory solution will be found by Kona or Cologne. There is a strong interest for the feature, but at the same time implementers insisted we need to tread carefully, containers already having huge overload sets.

## Merge source_location

Expect source_location in 20. The wait will prove worth it. In the end, source_location is mostly unchanged from the TS, except that now `source_location::current` is
an immediate function (`consteval`), so that you can not take its address. which is great, because that made no sense.

## Deprecate comma operator in subscript expressions


This paper managed to get consensus from Evolution so I expect it to go through core at Kona and hopefully be merged into the WD.
I'm hoping we will be able to have multidimensional subscript expressions in 23 maybe, 26 definitively.
Isabella Muerte presented some ways to be able to reclaim the `matrice[x,y]` syntax in the C++20 timeframe, we will have to wait to see if that can pan out.

{{< figure src="02.jpg" >}}

# More papers

## Relocation

I presented Arthur O'Dowyer's paper on relocation in terms of move plus destroyed. There was a very strong interest for the feature,
which will hopefully land in 23. I expect more work on that will be done in Cologne next summer. There are a lot of questions on how it affects
the memory model, but this is enough of a pain point that I am confidently hoping the committee will find a way to make it work.
[Arthur gave a CppCon talk about this proposal](https://www.youtube.com/watch?v=xxta6LEn9Hk) if you want to learn more.

We wondered whether we could work on a more general solution, namely, `destructive move` - although I'm not sure that would offer many benefits,
if at all, over what Arthur is proposing.

Orthogonally, EWGI discussed the possibility for `std::allocator` to support `realloc`, which should also improve the performances of vector
and string under specific workloads.

## optional<T&>

Sadly, `optional<T&>` died in a fire, which is a shame as I am afraid it will encourage people to use non-standard optional types.
Thanks, JeanHeyd Meneide for trying to make thing happen.
The best path forward might be to completely replace `std::optional` with a new type with more generic, better semantics. `std::maybe`? `std::box`?

## span

How many meetings does it take to write a type storing a pointer and a size? Quite a lot it seems.
We finally made span non-regular and fixed its signedness (that actually took a few sessions).
Expect a few more minor bug fixes in Kona.

This whole thing is probably a good case study for the wisdom of crowds.

Non-owning types are hard.

## Pattern matching

We had the first presentation on pattern matching. It looks great so far, C++23 will be the greatest release since C++20.
I tried to convince the committee we should preemptively reserve a keyword for this feature, alas nobody seems to see the need to do so.
`inspect` will probably fail to reach a consensus since it may break people's code. Brace yourself for `co_inspectexpr`.

## Format

A proposal based on the great `fmt` library (Victor Zverovich) has been accepted!
No IO _yet_, so its return must be fed to iostream for the time being.
But hopefully, we will be able to fix that soon-ish.
C++ is slowly turning into python, without python's performances. I'm very happy with that trend!

## Stacktrace

A library to print a stacktrace (authored by Antony Polukhin) is making its way to the wording group.
This is great because it needs compiler support to be implemented in a non-hackish optimal manner.

There are a lot more papers and feature to look forward in C++20 and C++23, from `flat_map` (Zach Laine) to a potentially lock-free concurrent queue
that can be used as a communication channel like in go (Lawrence Crowl).
Deducing this (Gašper Ažman, Simon Brand, Ben Deane, Barry Revzin) which is also an important feature, might make it to C++23.


{{< figure src="00.jpg" title="ABI break">}}

# Epilogue

This was my second meeting, and my first time on the US west coast. It was a blast.
The number of papers we looked at was truly astonishing.
As I struggle to recover from jet lag, I'd like to thanks all the people there, especially Tom Honermann who chaired the first official SG-16 meeting,
Bryce Adelstein Lelbach and JF Bastien who volunteered at the last minute to take on the very difficult job to chair the incubator groups
which were a huge success and were instrumental in ensuring the committee kept functioning smoothly - despite the surge of members and people - as well as the other chairs
and all the great people I got to meet there. And the scribes, the scribes, the scribes.

As I left the convention center, people told me "See you in Kona".

Qui vivra verra.

{{< figure src="05.jpg" title="Committee meeting" >}}



Oh and by the way, we merged ranges.





































