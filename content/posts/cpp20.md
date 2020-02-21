---
title: "Shipping C++20 in Prague"
date: 2020-02-19T09:21:00+01:00
---

{{< figure src="prague.webp" >}}


C++20 has shipped!

C++ is better and more alive than it has ever been.

You might have read on the Internet (rarely a good idea), that C++ grows too fast,
too complicated, too big.
I do not think this is true.

Bjarne Stroustrup reminded us that concepts are mentioned in [Design and Evolution of C++](https://www.amazon.com/Design-Evolution-C-Bjarne-Stroustrup/dp/0201543303), a book written in 1994, before even the first C++ standard. Coroutines and Modules are also old ideas that represent more than a decade of work. Ranges is probably the most refined large proposal and represent a huge body of work, notably from Eric Niebler and Casey Carter, with small tweaks from many, many people.
`std::fmt` also took many years of effort, while drawing from usage in other languages, notably Python.

This is not a coincidence:

I think most committee members are driven by the desire to make the language simpler.

Concepts may be hard to define, but they are easy to use. Same for ranges and views.
`std::format` is really easy to use, coroutines are challenging to write but easy to use and
make async code more maintainable and safer.

In all, C++ is gaining new tools to better express its core principles.

That doesn't mean C++20 is perfect or that no mistake was made.
But what one can consider a mistake is often the result of difficult to
balance trade-offs and well-understood compromises.
I might go into some of the things I don't like in a separate article,
if there is an interest for that.

Regardless, I think the entire committee tried to make
C++20 as best as we could and I think we were successful in that.
Objectively, C++20 is simpler and more expressive than previous C++ versions.


{{< figure src="river.webp" >}}


The release of C++20 means that features under the `-std=c++20` (or `-std=c++2a`)
flags of your compiler are now stable as far as the standard is concerned
and I would encourage you to use them as they become available in your compilers.
It will make you and your team more productive.
Of course, C++ is a tool: use what you need when you need it!

Modules do require build system support, I suspect this will remain
a mess for the foreseeable future.

Coroutines have no library components in C++20, you can use [cppcoro](https://github.com/lewissbaker/cppcoro) in the meantime.

Everything else can be used very simply as it becomes available in
compilers. We should also see an increasing number of talks, tutorials
and educative material on all these features.
You don't need to understand everything at once. It is a large release catering to
many domains, expert library writers and everybody else at the same time.

I have found that the small features are often the most immediately useful
and appreciated.
Things like [`.contains`](https://en.cppreference.com/w/cpp/container/map/contains),
[`ends_with`, `starts_with`](https://en.cppreference.com/w/cpp/string/basic_string/ends_with),
more optional `typename`, initializers for `if` and `range-for`, generated `=!`, spaceship,
etc

As is tradition, the room-by-room details of what happens in the committee
can be found on [Reddit](https://www.reddit.com/r/cpp/comments/f47x4o/202002_prague_iso_c_committee_trip_report_c20_is/).

As such, instead of trying to give an incomplete overview of the week,
I'd figure I should talk about what I have been working on these past few years.

## A busy couple of years

A little under two years ago, I went to my first meeting, in Rapperswil, Switzerland.
I don't remember exactly _why_, I guess I wanted to see how the sausage was made.

{{< figure src="sausage.webp" title="Sausage making" >}}

I have been to all meetings ever since, and have contributed the best I could to the sausage
making process, notably:

### Move-only iterators

[P1207](https://wg21.link/P1207)
[P1826](https://wg21.link/P1826)


Not all objects are regular. For example, file handles, sockets and coroutines handle are not
regular, which mean they cannot or should not be copied.

Iterators over these objects pretended to be regular because move-only objects were not a thing
when the STL was first standardized. This led to unsafe, less efficient code.

`std::ranges` allowed us to tweak the `iterator` concept to allow for move-only iterators.
This was a very small change to a core concept but it required a lot of work.
Would I have done it if I knew how much work it would require?
I don't know but I am sure glad I did.

### Pulling source_location out of the library TS

[P1208](https://wg21.link/p1208)

`source_location` is by large the work of Robert Douglas. It replaces the
__FILE__ and __LINE__ macros.
Unfortunately, it was slowly dying in the Library Fundamentals TS, as proposals in Library
Fundamentals do.
I convinced the committee to fish it out and merge it in C++20.
In the end, I got busy with too many things so I had to ask Robert to push `source_location`
through the finish line. It turns out he had to pull multiple all-nighters to redraft the wording several times... People pull heroics during meetings.

Fun fact, `source_location` is the first reflection facility merged into the standard and
the first (and so far only) `consteval` function in C++. Expect a lot more in 23!

### Deprecating comma operator in subscript expressions

[P1161](https://wg21.link/p1161)

I think a lot of people are excited about this one. It's the first step towards
a nice multi-dimensional indexing syntax, notably for `mdspan`.
I hope to have a proposal for that in Belfast, pending implementation.

Thanks to Isabella Muerte who had a [similar proposal](https://wg21.link/p1277r0)!

### Better constructors for span and string_view

[P1394](https://wg21.link/P1394)
[P1391](https://wg21.link/P1391)
[P1989](https://wg21.link/P1989)


`span` and `string_view` can now be constructed from a pair of contiguous
iterators.
`span` can additionally be constructed from any `contiguous_range`.
I was hoping to do the same treatment to `string_view` but because of the mess that is the `string` and `string_view` construction and conversion overloads, we decided to postpone that to 23.
I am hoping that this will get accepted in Varna, we will see.

### views::keys views::values views::elements

[P1035](https://wg21.link/p1035r7)

Christopher Di Bella did 99% of the work on these (and added a whole range of useful views).
At the names implies, `views::keys` and `views::values` let you iterate over the keys and values of an associative container.
`views::elements` is a generalisation of that: it let you iterate over the Nth elements of a sequence of tuples

## Some personal failures and rejected proposals

{{< figure src="glass.webp">}}

### ranges::to

[P1206](https://wg21.link/P1206)

`ranges::to` missed the train - We are hoping it will land early in C++23.
Many people have voiced their disappointment.
I'll try to provide a standalone header to do that, at some point.
One of the reasons it didn't land in 20 is that LWG was extremely busy and quite a few important
papers, including `stacktrace` are still in their queue.
The other is that I was unable to provide wording.
`static_extent` also missed the boat, I have no idea if we will be able to retroactively apply it to span.

### Make deprecated thing [[deprecated]].

[P1702](https://wg21.link/p1702r0)

LEWG decided they wouldn't want to force implementers to warn on depreciation,
which I think is unfortunate.
But it led to an interesting discussion about depreciation in the standard library,
so that paper was still very useful I think.

### inline in module

[P1604](https://wg21.link/p1604r1)

I failed to convinced the committee that `inline` in modules made no sense at all.
Fortunately, some of the damages were fixed by [ABI isolation for member functions - Davis Herring](https://wg21.link/p1779r3).
Unfortunately, `inline` still has too many meanings, especially in modules where it just shouldn't be a thing at all.

### module naming

[P1634](https://wg21.link/p1634r0)

Tooling rejected offering any kind of naming of structure guideline for modules.

[I believe](https://cor3ntin.github.io/posts/modules_naming/) this will have a long-lasting, negative effect on the ecosystem.
As a result of that, we can expect more build scripts, more extensions,
more incompatibilities between projects and overall, ever more brittle build systems.


## A great meeting

Our hosts Avast and Hana Dusíková were fantastic! They arranged for a barista to serve proper tasty coffee which was miles better than the usual conference "coffee". It's particularly appreciated in these meetings where many people have very little sleep.

{{< figure src="street.webp">}}

Prague turned out to be a wonderful city with lots of fun museums and great food!

## It takes an army

WG21 has now routinely well over 200 attendees and about 20 study groups.
It a lot of work by many, many very talented people over multiple years to build something
like C++20.
My own proposals were only possible thanks to the help of many people!

We have now turned our attention to C++23.

Independently of the so-called plan, I'm looking forward to reflection, sender-receivers, i/o,
rellocation, freestanding, more Unicode support, pattern matching, `std::embed` and many small quality of life improvements (including ~~ranges2~~ `ranges::to`, promise!).

See you in Varna!


{{< figure src="wg21.png">}}

