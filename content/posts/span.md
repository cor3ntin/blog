---
title: "A can of span"
date: 2018-05-14T12:36:14+02:00
---

The papers that will be discussed at the next C++ committee meeting [are out](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/#mailing2018-04).
The listing contains a number of interesting and controversial papers.
Among them, [Herbceptions](https://wg21.link/p0709), a number of concurrent concurrency proposals, a [proposal](https://wg21.link/P1063) calling for major design changes in the coroutines TS,
And an easy-to-review, [200 pages](https://wg21.link/P1037) long proposal to unify the Range TS in the `std` namespace.

In total, there are about 140 papers all rather interesting.

It's no wonder then that the hottest topic on the [Cpp Slack](https://cpplang.now.sh/) these past few days is `std::span`.

***Wait, what ?***

First off, if you are not on the Cpp Slack, you should, it's a great community.

Second, maybe you heard that `std::span` was already merged in the C++20 draft last meeting so why talk about it and why would a modest library addition make so much virtual ink flow?

Or maybe you never heard of `std::span` and are wondering what `std::span` even is.

{{< figure src="question.gif" title="Life, the universe and span" >}}

Trying not to break any eggs, I would say it can be described as **a fixed-sized, non-owning wrapper over a contiguous sequence of objects letting you iterate and mutate the individual items in that sequence**.

{{< ce >}}
#include <vector>
#include <gsl/span>
#include <iostream>

int main() {
    std::vector<std::string> greeting = {"hello", "world"};
    gsl::span<std::string> span (greeting);
    for(auto && s : span) {
        s[0] = std::toupper(s[0]);
    }
    for (const auto& word: greeting) {
        std::cout << word << ' ';
    }
}
{{< /ce >}}


This simply prints `Hello World` and illustrate the mutability of span's content.

`span` can represent any contiguous sequence, including `std::array`, `std::string`, `T[]`, `T* + size`, or a subset or an array or a vector.

Of course, not all containers are `span`, for example neither `std::list` or `std::deque` are contiguous in memory.

## Is span a view?

I'm not quite sure how to answer that. I wonder what the proposal says. So let's read the [span proposal](https://wg21.link/P0122):

> The span type is an abstraction that provides a **view** over a contiguous sequence of objects, the storage
of which is owned by some other object.

You might also have noticed that the paper is titled "**span: bounds-safe views**".

(Emphasis mine)

So a span is a `view`. Except it is called `span`. I asked around why was `view` called `span`, and the reason seems to be that the committee
was feeling like calling it `span` that day. In fact, when the span paper was first presented in front of the committee, it was called `array_view`.
An array in c++ being analogous to a sequence of contiguous elements in memory. At least, the vocabulary `Span` exist in `C#` with basically the same semantic.

### But now, we must talk about strings.

By that, I mean that we must talk about `std::string`.  For all intent and purpose, `std::string` is a `std::vector<char>`.
But people feel like strings are some special snowflakes that need their special container with a bunch of special methods.
So `string` gets to have a `length()` method because `size()` was probably not good enough for the princess, some `find*()` methods and lexicographical comparators.

And I mean, that's fair. A lot of application handle texts more than other kind of data, so having a special class to do so make total sense.
But fundamentally, the only difference between a vector and a string is that which is conveyed by the programmers' intent.

It should be noted that  `std::string` ( or `std::wstring` and the other `std::*string` ) is completely unsuited to handle text that is not encoded as ASCII.

If you are one of the 6 billions people on earth that do not speak English, you are gonna have a f��� bad time if you think `std::string` can do anything for you.
(Pardon my Chinese). At best, you can hope that if you don't mutate it in any way or look at it funny, it might still look okay by the time you display it somewhere.
That also includes the lexicographical comparators and the `find*()` methods. Don't trust them with text.

(Hang tight, the C++ committee is working hard on those issues!)

For the time being, it's best to see `std::*string` as opaque containers of bytes. Like you would of a vector.

Alas `string`, being the favorite child, got to have its own non-owning wrapper 3 years before anyone else. So in C++17, was introduced ~~`string_span`~~.
Nope, it is actually `string_view`.

It's a `string`, it's a `span`. It's the api of both mixed together. But it's called a `view`.

It has all the same special snowflakes methods that `string` has.

I am mean, those methods aren't that bad. The author of the [`string_view` paper](https://wg21.link/n3921) had some very nice thing to say about them:

> Many people have asked why we aren't removing all of the find* methods, since they're widely considered a wart on std::string.
> First, we'd like to make it as easy as possible to convert code to use string_view, so it's useful to keep the interface as similar as reasonable to std::string.

There you have it : a backward compatibility wart.

So, maybe we could actually define `std::string_view` in term of `span`?

```cpp
template <typename CharT>
class basic_string_view : public std::span<CharT> {
    std::size_t length() const {
        return this->size();
    }
};
```
Simple and easy!

**Except** this is completely wrong because unlike span, `std::string_view` is a **non-mutable** view.

So it is actually more like more like

```cpp
template <typename CharT>
class basic_string_view : public std::span<const CharT> {/**/};
```

Going back to the [`string_view` paper](https://wg21.link/n3921), the author explains that:

> The constant case is enough more common than the mutable case that it needs to be the default.
> Making the mutable case the default would prevent passing string literals into string_view parameters, which would defeat a significant use case for string_view.
> In a somewhat analogous situation, LLVM defined an ArrayRef class in Feb 2011, and didn't find a need for the matching MutableArrayRef until Jan 2012.
> They still haven't needed a mutable version of StringRef. One possible reason for this is that most uses that need to modify a string also need to be able to change its length,
> and that's impossible through even a mutable version of string_view.

It is hard to argue with that, especially given what I just said about strings. So `basic_string_view` is non-mutable because it's a sensible default _for strings_.

> We could use typedef basic_string_view<const char> string_view to make the immutable case the default while still supporting the mutable case using the same template.
> I haven't gone this way because it would complicate the template's definition without significantly helping users.


However, C++ is mutable by default, and constness is opt-in.
So having a type being `const` by default, while more appealing to our modern, wiser sensibilities may not be that great: there is no way to opt-out from `basic_string_view` constness.
Since `mutable` always is the default, the language does not provide a way to construct a `basic_string_view<mutable char>`.


Special snowflake methods aside, there is 0 difference between `typedef basic_string_view<const char> string_view` and `basic_string_view : public std::span<CharT>`.
So, `std::span` is a view, `std::view` is a span, both classes are basically the same thing and have the same memory layout.

So much similar in fact that [a brave soul suggested that they could be merged](https://wg21.link/P0123).
That was back in 2015 when `span` was still called `array_view`.

Unfortunately, some people now think the term `view` somehow implies immutable.

But the only reason one might think so boils down to `string` hijacking a vocabulary type all for itself.
And guess what's the last thing you should do to a utfX encoded string? Randomly slicing it into views at code unit/bytes boundary.

In the *Ranges TS*, nothing either implies views are immutable:
> The View concept specifies the requirements of a Range type that has constant time copy, move and assignment operators;
> that is, the cost of these operations is not proportional to the number of elements in the
View.

TL;DR: view and span: same thing; `string_view`: special confusing little snowflake.

Moving on...


## Is span a range?

In C++20, a range is very simply something with a `begin()` and an `end()`, therefore a `span` is a range.
We can verify that this is indeed the case:

{{< ce_fragment compiler="gcc-concepts" >}}
{{< ce_hidden >}}
#include <stl2/detail/range/concepts.hpp>
#include <vector>
#include <gsl/span>
{{< /ce_hidden >}}
{{< ce_code >}}
static_assert(std::experimental::ranges::Range<std::vector<int>>);
static_assert(std::experimental::ranges::Range<gsl::span<int>>);
{{< /ce_code >}}
{{< /ce_fragment >}}

We can refine that further, `span` is a **contiguous range**: A range whose elements are contiguous in memory.

While currently neither the notion of `contiguous iterator` or the `ContiguousRange` concept are part of C++20, [there is a proposal](https://wg21.link/P0944).
~~Weirdly, I could not find a proposal for `ContiguousRange`~~[^fix_contiguous_range]. Fortunately, it is implemented in `cmcstl2` so we can test for it.


{{< ce_fragment compiler="gcc-concepts" >}}
{{< ce_hidden >}}
#include <stl2/detail/range/concepts.hpp>
#include <gsl/span>
{{< /ce_hidden >}}
{{< ce_code >}}
static_assert(std::experimental::ranges::ext::ContiguousRange<gsl::span<int>>);


{{< /ce_code >}}
{{< /ce_fragment >}}

So, given that we know that `span` is basically a wrapper over a contiguous range, maybe we
can implement it ourselves?

For example, we could add some sugar coating over a pair of iterators:

{{< ce compiler="gcc-concepts" >}}
#include <gsl/span>
#include <stl2/detail/range/concepts.hpp>
#include <vector>

template <
    std::experimental::ranges::/*Contiguous*/Iterator B,
    std::experimental::ranges::/*Contiguous*/Iterator E
>
class span : private std::pair<B, E> {
public:
    using std::pair<B, E>::pair;
    auto begin() { return this->first; }

    auto end() { return this->second; }

    auto size() const { return std::count(begin(), end()); }

    template <std::experimental::ranges::ext::ContiguousRange CR>
    span(CR &c)
        : std::pair<B, E>::pair(std::begin(c), std::end(c)) {}
};

template <std::experimental::ranges::ext::ContiguousRange CR>
explicit span(CR &)->span<decltype(std::begin(CR())), decltype(std::end(CR()))>;

template <std::experimental::ranges::/*Contiguous*/Iterator B,
          std::experimental::ranges::/*Contiguous*/Iterator E>
explicit span(B && e, E && b)->span<B, E>;

int main() {
    std::vector<int> v;
    span s(v);
    span s2(std::begin(v), std::end(v));
    for (auto &&e : s) {
    }
}
{{< /ce >}}

Isn't that nice and dandy?

Well... except, of course, this is not a `span<int>` ***at all***. It's a freaking

```cpp
span<
    __gnu_cxx::__normal_iterator<int*, std::vector<int>>,
    __gnu_cxx::__normal_iterator<int*, std::vector<int>>
>
```

Pretty pointless, right?

See, we can think of `views` and `span` and all those things as basically "template erasure" over ranges.
Instead of a representing a range with a pair of iterators whose type depends on the underlying container,
you'd use a view/span.

However, a range is not a span. Given a `ContiguousRange` - or a pair of `contiguous_iterator`,
it is not possible to construct a `span`.

This will not compile:

{{< ce_fragment >}}
{{< ce_hidden >}}
#include <vector>
#include <gsl/span>
{{< /ce_hidden >}}
{{< ce_code >}}
int main() {
    constexpr int uniform_unitialization_workaround = -1;
    std::vector<int> a = {0, 1, uniform_unitialization_workaround};
    gsl::span<int> span (std::begin(a), std::end(a));
}
{{< /ce_code >}}
{{< /ce_fragment >}}

So on the one hand, `span` is a range, on the other, it does not play well with ranges.
To be fair, `span` was voted in the draft before the great [Contiguous Ranges](https://wg21.link/P0944) paper could be presented.
But then again, that paper hasn't been updated afterwards, and Contiguous Ranges have been [discussed since 2014](https://wg21.link/n3884), including by the
string view paper.

Let's hope that this will be fixed before 2020!

In the meantime, using span with the std algorithms will have to be done like that I guess.

{{< ce_fragment >}}
{{< ce_hidden >}}
#include <vector>
#include <gsl/span>

int main() {
    std::vector<std::string> names {
        "Alexender", "Alphonse", "Batman", "Eric", "Linus", "Maria", "Zoe"
    };
{{< /ce_hidden >}}{{< ce_code >}}
    auto begin = std::begin(names);
    auto end = std::find_if(begin, std::end(names), [](const std::string &n) {
                    return std::toupper(n[0]) > 'A';
    });
    gsl::span<std::string> span {
        &(*begin),
        std::distance(begin, end)
    };
}
{{< /ce_code >}}
{{< /ce_fragment >}}

Which is nice, safe and obvious.

Because we are speaking about contiguous memory, there is an equivalent relationship between a pair of
`(begin, end)` pointers and a `begin` pointer + the size.

{{< figure src="drake.jpeg" title="Ranges all the things" >}}

Given that, we can rewrite our span class

{{< ce_fragment compiler="gcc-concepts"  >}}
{{< ce_hidden >}}
#include <gsl/span>
#include <stl2/detail/range/concepts.hpp>
#include <vector>
{{< /ce_hidden >}}{{< ce_code >}}
template <typename T>
class span : private std::pair<T*, T*> {
public:
    using std::pair<T*, T*>::pair;
    auto begin() { return this->first; }

    auto end() { return this->second; }

    auto size() const { return std::count(begin(), end()); }

    template <std::experimental::ranges::ext::ContiguousRange CR>
    span(CR &c)
        : std::pair<T*, T*>::pair(&(*std::begin(c)), &(*std::end(c))) {}

    template <std::experimental::ranges::/*Contiguous*/Iterator B,
              std::experimental::ranges::/*Contiguous*/Iterator E>
    span(B && b, E && e)
        : std::pair<T*, T*>::pair(&(*b), &(*e)) {}
};

template <std::experimental::ranges::ext::ContiguousRange CR>
explicit span(CR &)->span<typename CR::value_type>;

template <std::experimental::ranges::/*Contiguous*/Iterator B,
          std::experimental::ranges::/*Contiguous*/Iterator E>
explicit span(B && b, E && e)->span<typename B::value_type>;
{{< /ce_code >}}{{< ce_hidden >}}
int main() {
    std::vector<int> v;
    span s(v);
    span s2(std::begin(v), std::end(v));
    for (auto &&e : s) {
    }
}
{{< /ce_hidden >}}
{{< /ce_fragment >}}

This behaves conceptually like the standard `std::span` and yet it is easier to understand and reason about.

### Wait, what are we talking about? I forgot...

```cpp
template <typename T>
struct {
    T* data;
    std::size_t size;
};
```

Oh, right, freaking `span`!

I guess that my point is that `contiguous ranges` are the general solution to `span`. `span` can be easily described in term of a contiguous range.
Implementing or reasoning about `span` without `contiguous ranges` however is trickier.
`string_view` being a further refinement on span, it's clear that the committee started with the more specialized solution
and is progressing to the general cases, leaving odd inconsistencies in its wake.

{{< figure src="vasa.jpg" title="Death by a thousand paper cuts" >}}

So far we established that `span` is a view by any other name and a cumbersome range. But, what's the actual issue?

## Something very, very wrong with span

I would go so far as to say that `span` (and `view`, same thing) breaks C++.

The standard library is built on a taxonomy of types and particularly the concept of a `Regular` type.
I would not pretend to explain that half as well as Barry Revzin did, so go read
[his great blog post](https://medium.com/@barryrevzin/non-ownership-and-generic-programming-and-regular-types-oh-my-d35cd490d402)
which explain explains the issue in detail.

Basically, the standard generic algorithms make some assumptions about a type in order to guarantee that the algorithms are correct.
These type properties are statically checked at compile time, however, if a type definition does not match its behaviour, the algorithm will compile
but may produce incorrect results.

Fortunately, span is the definition of a `Regular` type. You can construct it, copy it around and compare it. So it can be fed to most standard algorithms.
However, comparison operators don't actually compare two `span`, they compare the data `span` **points to**. And as Barry illustrated,
that can easily lead to incorrect code.

Tony Van Eerd having a knack for distilling fundamental truths,
observed on slack that while the definition of `Regular` was quite precise (but, as it turns out, not quite precise enough to handle `struct {T* ptr };` ),
its intent was to guarantee that handling `Regular` objects should have no effects on the rest of the program. Being proxy objects, `span` defy that expectation.

On the other side of the table, users of the STL can reasonably expect `span` to be a drop-in replacement for a `const vector &`.
And that happens to be mostly the case, you can compare it to a vector, iterate over it...
Until of course, you try to copy it or change its value, then it stops acting like a `vector`.

# Unmet expectations

`span` is a `Regular` type. `span` is a pointer to a chunk of memory. `span` is a value. `span` is `SemiRegular`, not `Regular`.
`span` looks like castor and bites like a snake, but is actually a duck-billed platypus, a monstrous hybrid that foils every attempt at
classification.

`span` has a dual nature, an irreconcilable ambivalence that gets half the committee scrambling hopelessly to find some form
of solace in the teachings of Alexander Stepanov while the other half has been caught whispering that maybe we should rewrite everything in rust.

> Can you freaking stop with the lyrical dramatisation?

Hum, right. Sorry.


But really, `span` tries to please both library writers as to be well behaving in generic algorithms and non-library writers as to offer a nice, easy to use API.
Noble goals indeed.

However, you can't have your cake and eat it too. And so is span bad at being a container proxy and bad as being a well-behaved standard `Regular` type.
By its dual nature, its api is easy to misuse and its humble appearance make it look like an innocent container-like thing rather than the deadly trap it is.
It stands to reason that **if the API is in any way easy to be misused, it will be**. And so `span` is nothing but an unassuming foot-blowing nuclear warhead.


In short, it does not meet expectations, because some of its design goals are antithetical. Specifically:

 * It's a pointer-like object whose comparison compares the content of the underlying data.
 * It's a container-like object whose assignment does not actually change the underlying data.


# Fixing span

*Can such a monster even be tamed ?*

I believe it can, and it actually would not require much.

There is in fact nothing intrinsically wrong with `span`, we just need it to drop the mask and be upfront about its true nature.
A lot can be said about the importance of naming things properly, and as far as `span` is concerned, there are more than a few names wrong.

Let's unpack

## span::operator==()

There are whole fields of mathematics dedicated to describing how things are "equal" or comparable.
Careers were made, books were written, libraries filled, it was theorized, organized, researched, and ported to Haskell.
That's why, in its infinite wisdom, `perl 6` dedicated a few tokens to describe the equality of things:

```perl
==
eq
===
aqv
=:=
=~=
~~
```

Meanwhile, `std::span` is collapsing the entire group theory into 2 characters.
And of course, there is only so much meaning one can imbue to a 2-byte token.

A lot of arguing between committee members has been about whether `operator==`
should compare the identity (whether two span points to the same underlying data),
or the elements.

There are supporters of both meanings, and they are both ~~wrong~~ right.
No really, I believe they are _wrong_. (I'm gonna make so many friends with that article...).

If both sides of the argument make as much sense as the other, it's because there isn't an answer.
It starts to be about made up arguments to back up one's personal preferences which usually is somewhere between those two extremes:

 * We should abide by the type categories and the standard library correctness otherwise we will inevitably blow our own foot.
 * We should meet user expectations otherwise they will blow their foot and then have our heads.

Both of which are very right and sensible positions to hold and respecting both these point of views is necessary.

The only way to avoid a bloodbath is to, therefore,**remove completely all comparison operators**.
If you can't compare them, you can't compare them wrongly.

Unfortunately, if a type isn't comparable, the `stl` kinda stop working - the type stops being `Regular` and concretely sort and search algorithms will not work.

A solution may be to resort to some `ADL` trickery to make `span` comparable only in the context of the standard library.
That can be demonstrated:

{{< ce >}}
#include <vector>
#include <algorithm>

namespace std {
    class span { };
}

namespace __gnu_cxx::__ops {
    bool operator<(const std::span &a, std::span &b);
}

void compile() {
    std::vector<std::span> s;
    std::sort(s.begin(), s.end());
}

//void do_no_compile() {
//    std::span a, b;
//    a < b;
//}
{{< /ce >}}

That would make `span` truly regular within the stl, and prevent people comparing the wrong thing.
Element-wise comparison would be done through `std::equal`.


## span::operator=()

Depending on whether span is seen as a pointer or a container, one could assume that  we are setting the span pointer or
the underlying data;
Unfortunately, we can't use the same ADL trick as for `==`, and I don't see any other reasonable solution.
There is another way we can fix `operator=` though:  By making it very clear than span behaves like a pointer...

## Renaming span

`span` used to be called `array_view`. It's easy to see a `view` as a pointer (not in the context of the range TS though).
`view` makes it extra clear that it's a view and therefore non-owning.

`array` carries that it is a pointer to a contiguous memory segment because that's what arrays are in the C memory model.

And yeah, that would mean that `array_view` is mutable and `string_view` is constant.

It makes no sense. However, it makes a lot more sense than having a very confusing `span` type that the world's best experts are
not quite sure what to make of.

## It does not stop there...

A couple of papers were publishing, relieving more issues with span
* [Its size is, for some reason, signed] (https://wg21.link/p1089)
* [Its API has some inconsistencies] (https://wg21.link/p1024)

## Changing people?
Some believe that we should teach people that platypus are ducks because that sure would be convenient.
But, while meeting expectations is hard and sometimes impossible, trying to make people change their expectations completely
sounds a bit unreasonable to be. At best it takes decades, and by the time the collective knowledge and wisdom starts shifting, the
experts on the front lines will need people to have a completely new set of expectations.

Sure, sometimes nothing can replace education, talks and books. However, teachers have bigger battles to focus on than `span`.

## A simpler story for views and ranges

After having sorted the mammals on one graph and the birds on the others, I imagine biologists were pretty pissed to see a flying squirrel.

However, the committee is not just classifying existing types they are designing them.
And I wonder if - as much fun as it may be to see them jumping over the canopy - we actually have a need for non-mutable flying squirrels.

 * `Ranges` are... ranges represented by a pair of iterators. Either owning(`Containers`), or non-owning(`Views`)
 * `Views`  are... non-owning views over ranges.
 * `array_view` and `string_view` offer erasure of a view over a range represented by a pair of iterators who happen to be pointers.
 * Containers own data

Maybe that's not quite accurate. But we need an unifying theory of everything.


To conclude this short introduction of `span`, I will leave you with this photo of a giraffe.

{{< figure src="horse.jpg" title="Span" >}}


[^fix_contiguous_range]: I originally mentioned incorrectly that `ContiguousRange` was not proposed for inclusion in the C++ Standard. [This is incorrect](https://wg21.link/P0944)
