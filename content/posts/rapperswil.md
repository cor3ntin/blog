---
title: "Rapperswil Committee Meeting: A Trip Report"
date: 2018-06-11T12:26:11+02:00
---

{{< figure src="04.jpg">}}

This was my first committee meeting.
I arrived Sunday morning at Jona, the next town over where I had a lovely AirBnB in a very nice, peaceful suburb.
I settled to visit Rapperswil but met some people from Nvidia going to the meeting. So we naturally started to talk about C++.
The meeting was to last 6 days and until the very end, we talked about C++, every minute of every hour.
Needless to say, I had no trouble falling asleep, usually around 1 AM.

I was not presenting any paper, nor taking notes, yet I struggle to recover from this very taxing week.
Papers authors sometimes had to work a few more nightly hours to tweak some wording.
And I have a feeling that the working-groups chairs had it even worse.

Yet, it was a wonderful, enlightening experience!

Most importantly, I got to meet a lot of cool people working in all sort of fields I barely knew existed.
It was interesting to see how much people could have very opposite views and strongly disagree
with one another during sessions and still be friends and have a drink afterward.

Everyone there deeply cared to make C++ better, even though we might have had very different opinions on what "better" is
or what the best way to go about things is. And after that meeting, I don't think the idea that committee members make the "worse compromises"
holds much water. In most case, people were genuinely trying to understand and accommodate the use case and point of views of other people.

It's also clear to me that, for better or worse, who is in the room does matter, a lot.
For example, I had not much love for `is_constant_evaluated` that was discussed on Friday, and probably would have voted against if not for that
one comment that changed my opinion to "strongly in favor".
Maybe I influenced a few people, who knows? I definitively inflicted my terrible French accent and for that I'm sorry.

Consistency was discussed much more than I thought it would. Of course, people don't share the same definition of what consistency is or should be and,
that too depends on who is in the room. But we try.

It's already a bit fuzzy in my head, and [Bryce already made a great summary on reddit](https://www.reddit.com/r/cpp/comments/8prqzm/2018_rapperswil_iso_c_committee_trip_report/),
but I will try to go over some things that happen in the week.

Of course, LEWG, EWG, Core, and LWG meet at the same time, some others SG meet too, so it's not possible to assist to everything at once. Which is a bit annoying because often,
a lot of things happen at the same time. And the schedule is fuzzy and flexible, to say the least (It's also incredibly optimistic).
I elected to spend my time between EWG and LEWG because I care more about new exciting features than wording. This is going to bite me when I have to write a paper.

### Monday
was not off to a great start. We took two hours to decide `basic_string_view(nullptr)` should be UB. Bummer. I hate views.
We then decided to merge feature macros into the working draft, a decision I'm very happy about. They are very useful if you need them and don't hurt if you don't.
Of course, we don't quite know how that would work in a module-only world, but this answers a concrete problem until we hopefully have something better.

We then fixed initialization and aggregates. Ok, we only fixed a tiny part of it, by reverting to C++98 rules for the definition of what an aggregate is.

```cpp
struct aggregate {};
aggregate a{};

struct not_aggregate {
    not_aggregate() = delete;
};
not_aggregate b{}; //ill formed


struct not_aggregate2 {
    not_aggregate2() = default;
    int a;
};
not_aggregate2 c{42}; //ill formed
```

It is interesting that this change was very well received despite being a small breaking change.

Competing proposals were deemed too complicated and it was refreshing to see that things have been made simpler, for once.

After dinner, we talked about concept syntax... this warrant a dedicated blog post, stay tuned!

### Tuesday

I spent the morning at EWG talking about modules.
It was a very constructive morning about a conjoint proposal from the ATOM proposal and TS authors.

I am mostly happy with the current state of affairs even if I still don't see the added value of partitions.
I'm concerned about lexing and preprocessing being increasingly more of a tangled mess but the solution seems unusable by tooling.
I really hope they make it to C++20.
Unfortunately, I did not stay there in the afternoon so I don't know if exporting macros were discussed. This would be kind of an over-my-dead-body thing.

I elected to spend the afternoon in LEWG instead where we did the work of going through all the 190 pages of the Ranges TS and decided to forward it to LWG,
the next step before being merged into the working draft.

{{< figure src="02.jpg">}}

### Wednesday

We discussed the creation of a new data persistence Study Group and enough people were interested that this might formally happen in the coming months.
This group would focus on bringing low level i/o facilities such as async files and memory mapped files.

I raised the concern that it should be separated from text formatting and people seem to be in agreement. So this work might lead to facilities saner than `iostream`,
on top of which others could build separate formatting and localization facilities. Nothing not to love!

Speaking of text formatting, Victor Zverovich presented an update to his `fmt` proposal, and we decided this is a high priority item that LEWG should focus on in the C++20 time frame.
I'm very happy about that. The alternative was to send it to a TS (library extensions v3) and given all the great things in
[ibraries extensions v2](http://en.cppreference.com/w/cpp/experimental/lib_extensions_2) that are not merged into the WD, I was concerned we might not have `fmt` any time soon.

I went to part of the `coroutine` talks. It was... entertaining. I have no expert opinion on this subject, however, I think the current TS is the good level of abstraction for most users
and the point was made that core coroutines could be a subset of future coroutines implementation rather than an either-or question.

Coroutines got consensus in EWG but on Saturday they did not reach consensus.

In LEWG, Eric Niebler presented his "Deep integration" proposal for ranges, with much applause.
So the rangified algorithms will live in `std::ranges` while the iterators and traits will remain in `std` avoiding a lot of complexity and duplication.
We also approved `ContigousRanges`, which is a small but very exciting refinement of ranges and iterators!


After dinner, the Direction Working Group gave a detailed presentation of
[p0939r0](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0939r0.pdf). [Bjarne talked about the Vasa](http://open-std.org/JTC1/SC22/WG21/docs/papers/2018/p0977r0.pdf).

I'm still unsure what teachings extract from all of that. And the tale of the Vasa can be interpreted in a lot of different ways.
We mostly agreed that this was all very complicated and that there were no right answers.

We discussed the usefulness and nature of TS, there again without a clear outcome.

We, however, reached the agreement that all good papers should have a rationale, an abstract, a quote and a Tony table.

I left the room with more question than I had going in.
Later that week I asked some people whether WG21 could benefit from better tooling to manage, search and keep track of papers.
If that's something that interests you, I really think a lot can be done in that area.

(The currents tools are in my opinion, quite unsatisfactory)

{{< figure src="05.jpg">}}

### Thursday

Thursday was a good day.
We started off by making `std::span` `SemiRegular`, thereby restoring order in the universe. A poll was taken on whether we should nuke span altogether and that, unfortunately, did not reach consensus.
It was noted that `std::ranges` has a `subrange` type that does most of what span does, except better.
But, realistically, I think that removing all comparisons operator was the best outcome we could hope for and I'm really glad we agreed on that. Thanks, Tony.
Let's hope _the room_ does not change its mind at the next meeting.

Making `span` span unsigned was not as clear-cut, but the general consensus was to make span consistent with existing containers and find a more general solution towards signed containers.

Herb presented part of the static exception proposal and we almost unanimously decided that we did not care about `std::bad_alloc` going forward.

I skipped a few session to talk about dependency management with some people and just try to rest my brain a bit.

After dinner, we had a long conversation about 2D Graphics.
I'm a bit confused about the actual outcome of that evening and [Guy already did a great summary from his point of view](hatcat.com/?p=48).
I think it's dead, Jim. I think it's a good thing. Nonetheless, we all expressed that what we want, what we really really want, is a dependency manager.

### Friday

The proposal for a stacktrace library went well.
I'm very excited about that. There are a lot of libraries out there to do that but the compiler has opportunities to do much better.
I think it will be useful to a lot of people (including me !). The API is small and clean, and I have good hope it will be in the C++20 IS.

LEWG also rangify some uninitialized memory algorithms I did not know existed.
And noted it should rangify more stuff. Brace yourself for the "Rangify all the things" era. I am definitively not complaining.

I then went back to EWG were we talked about `constexpr!` and `std::is_constant_evaluated()`, both of which got the favors of the crowd.
`constexpr!` is particularly great for reflection and [`std::embed`](https://wg21.link/p1040). in short `constexpr!` is always constexpr.

Unfortunately, we did not get to bikeshed the name. A proposal for `true constexpr` was rejected. I think it's a better choice, despite looking completely ridiculous at first.
I wish we would not have hijacked the `!` so lightly.

Some more `constexpr` things were voted in while I was not in the room.
The committee is working toward making `std::string` constexpr and toward a value-based concept syntax. Which is great if you ask me!

Unfortunately, `Deducing this` was discussed while I was not in the room. It apparently did not get much love in its current form. And the great 'adjective'-like syntax was, quite unfortunately bikeshed against. Sad face.
I hope it comes back in a slightly different form, maybe unified with the `unified function call` proposal.

{{< figure src="03.jpg">}}


#### I went to the tooling evening session with very little expectations and hope.

Boris presented `build2` then Titus gave a presentation about refreshed long time goals.
It was less disorganized than I feared and we seem to all agree on the general direction we should go towards.
We had a fruictiful discussions regarding particular subjects (source vs binaries, diamond dependencies, etc)...
And we definitively want a dependency management system in some form.
Nothing positive was said about CMake, it's a start!

The lengthy conversations I had with the main developer of `vcpkg` certainly makes me want to use `vcpkg` more.

Wait and see, I guess?

### Saturday

We took official votes on Saturday, it went quite well!
Being a member of the French NB, I got to vote and I don't remember opposing anything.

But the vote for merging the Coroutines TS did not go well and did not reach consensus.
The convener proceeded to take another vote but this time it was a single vote per country - that is I think 11 votes total.
It failed to reach consensus again.

I know it's part of the ISO process but to me, it seems really odd and broken to involve "countries" in that kind of things, especially given that
most countries are very small national bodies (1-5 people).

I guess it's the worse system, except all others?

Most other motions passed and I think I was satisfied with everything,
showing that the work done by the working groups is very effective at producing features that gather consensus.

After launch, I got back to a LEWG to squeeze a few more papers through.
But by that time I was really exhausted and a one-hour long discussion on memory alignment did not help.
Nevertheless, we agreed we wanted `std::asume_aligned` if for nothing else than to replace a whole bunch of ugly macros and intrinsics.

Someone presented a very clever but ungodly way to have some sort of lifetime extensions for passing rvalue-ref strings to a function taking a string view.
The nastiness of it put me off, but I learned a few things about lifetime extension.

Earlier in the week, someone presented a language-level lifetime extension mechanism which seems like a more reasonable approach to me.
Some kind of Rustification of C++ memory model would not hurt, especially given that ranges and Rvalue-refs don't work well together at all.

After that, we discussed a coupled more papers but I was out of it and started to make some very stupid comments.
People left one after the other until we did not have quorum. It was time to adjourn.


### Unicode
Missing a chair, SG16 did not officially meet, however half of us were here and some papers made great progress (I was not involved in that amazing effort).


### The way back
I made the unfortunate decision to not travel back from Zurich and found myself somewhere deep in the French countryside Saturday evening.
Unable to find a public transport or book an Uber, My Airbnb host was kind enough to dig me out of the hole I made for myself and drove me to the airport at 5 AM.
A few hours later I was home.  Alas, LEWG is a treacherous place so I brought home some homework.


I got to see how the sausage is made and even if a lot of it could be certainly be improved (somehow),
I am looking forward to the next meeting - which will probably be Cologne in 2019 for me, even if I kinda want to go to San Diego...

All the people I met were great and I gained a new respect for how C++ is made.

I went to Rapperswil with a lot of fears and apprehensions (mostly feared to be too stupid to be there, which definitively was the case at times, but being the stupidest person in the room is humbling !),
but in the end, it was a very satisfying, positive experience.

{{< figure src="01.jpg">}}

See you next time!
