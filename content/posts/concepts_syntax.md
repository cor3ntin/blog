---
title: "The tightly-constrained design space of convenient template functions declaration syntaxes"
date: 2018-06-13T11:32:09+02:00
draft: true
---

Did you know that the concept TS was merged into the Working Draft in July 2017, in sorry Toronto?
And we are a plank length away from merging the Range TS in C++20 as well, including a few goodies such as projections,
contiguous ranges/iterators and ranges adaptors?
We also added a bunch of general-purpose concepts in the `std` namespace in Rapperswil.

Concepts were 3 decades in the making and the range TS is a huge body of work.
Yet, I feel like a lot of people are unaware of these great features that are coming to a compiler near them.

It might be that only GCC has an implementation of concepts (that do not quite matches the TS and that is easily tripped), making experimentation a bit hard.
Or maybe people are tired of waiting? After all, we were promised ~jetpacks~ concepts in C++11, C++14, C++17.

Or maybe the expert-friendly syntax of concept usage scares people away?

# What are concepts?

Truth is, there is little concepts can do that can not already be achieved with C++17 and (a lot) of SFINAE.
Eric Niebler's widely popular `ranges-v3`, which was the foundation of the Ranges TS makes heavy use of ""concepts"" using a lot of
[SFINAE tricks and a few macros](http://ericniebler.com/2013/11/23/concept-checking-in-c11/). And honestly, using `ranges-v3` to define or
refine your own concepts is rather easy.
Still, without a lot of metaprogramming tricks than most developers should not be expected to fully understand, SFINAE is tricky and error-prone.
Concepts aim to provide an easy way to describe complex requirements on individual types and advanced overload sets.

The second thing that concepts offer is better error messages (even if this is, of course, a quality-of-implementation matter).
The compiler can pinpoint exactly what requirement(s) a type is missing for a given template instantiation but it can't
know which template you were trying to instantiate as it can't read your mind to resolve ambiguities. [Yet](https://www.aaai.org/ocs/index.php/AAAI/AAAI17/paper/view/14603).

It's probably [less magical than what you might expect](https://godbolt.org/g/g8gfLM), so it will not absolve
C++ developers to be able to understand cryptic compilation errors spawned somewhere inside a deep template instantiation stack,
however to an experienced developer, the errors will be much more explicit.

So, it does not seem not too inaccurate to view Concepts as sugar coating over SFINAE, with the added bonus of more explicit errors messages.
It may not seem very exciting, admittedly.

But since Bjarne Stroustrup dreamed concepts a few things happened.
First, meta-programming tricks, knowledge, and libraries, as well as more robust compiler implementations enabled libraries such as ranges-v3 to exist.
At the same time, the Concepts proposals were simplified to the point where concepts as they were merged used to be called "concepts-lite",
dropping both [concepts maps](https://isocpp.org/wiki/faq/cpp0x-concepts-history#cpp0x-concept-maps) and [axioms](https://isocpp.org/wiki/faq/cpp0x-concepts-history#cpp0x-axioms).

Yet concepts set to achieve a very important goal:
Bridging the gap between Imperative programming and Generic Programming, by making templates easier to use and seamlessly integrated.
Generic programming would then be more easily accessible to most non-experts C++ developers and it would then be easier to write inter-operating libraries. Reusable, modular, explicit APIs.

There was, however, an issue.
Templates were always somewhat unfriendly to non-experts, and adding a bunch of `requires requires` clauses in the mix did not improve the situation.

{{< youtube id="H8HplZtVGT0" autoplay="false" >}}


# Short syntaxes

To make concepts more palatable, the Concept-Lite proposal (circa 2013) introduced a bunch of shorthand syntaxes.

```cpp
template<typename T>
concept Foo = true;

//template introducer syntax.
Foo{T} void foo(const T&);
//abbreviated function syntax
void bar(const Foo&);
//abbreviated function syntax, auto being the least constrained possible constraint
void bar(auto);
```

And it was easy, it was reasonably elegant, and everything was well in the world.
But then, questions arise, concerns were raised:

What about the meaning of multiple parameters constrained by the same concepts?
How distinguish generic functions from the non-generic ones? What about collapsing universal forwarding references?

As the ink was flowing, C++14 shipped. As C++17 sailed, defenders and detractors of the concept abbreviated syntax dug trenches until progress on the Concept TS
came to a gloomy stop.

In this context, a brave soul suggested that we could maybe remove the abbreviated syntaxes from the TS and merge the non-controversial bits in the TS.
And for a little while, a truth was established, allowing concepts to be merged in the Working Draft, while Tom Honnerman enjoyed his well deserved moment of glory.

**However**.

The committee still wanted a ~~short~~ ~~abbreviated~~ ~~terse~~ ~~natural~~ convenient syntax. They just could not agree on which syntax was best. It was back to the drawing board.


You might think that getting consensus on syntax would be easier.
It turns out the design space is ridiculously complicated, so let me try to describe some of the numerous constraints.


# The Design Space

## 0 - The meaning of `void f(ConceptName a, ConceptName b)`

Up until last year, some people argued that given `void f(ConceptName a, ConceptName b)`, `a` and `b` should resolve to the same type.
Fortunately, [this issue was resolved](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0464r2.html),
and there is now a tacit(?) consensus that each parameter should be deducted separately and be of potentially different types.

So, in fact, _some_ progress was made on the convenient syntax and things are moving in the right direction


## 1 - It's purely a syntax issue.

* Concepts are in the WD
* Every imaginable set of constraints can be applied to type and methods using the non-convenient syntax as per the WD
* The compiler (and by extension, tools) needs no syntax whatsoever to distinguish concepts, types, values, type-concepts, value-concepts.
* There may be some questions regarding references but the same solution can be applied regardless of the actually chosen syntax
* The question then is what the best way to please the finicky human developers might be.

## 2 - Simple and natural

The main goal being making templates simpler for most people, we need to find a simple syntax.
Something easy to teach and easy to learn.
Hopefully intuitive. But people intuitions change.
People have different expectations depending on their background, the other languages they, there skill level with C++...
It's to be expected that a given syntax will be intuitive to the author that proposed it and most experts will eventually grasp almost anything.
So what is simple is highly debatable and subjective.

But we can set up a few guidelines

* Not too verbose: Syntaxes that require putting a large number of tokens in a particular order are often hard to grasp
* Not too alien: We can look at other languages too see how concepts could be expressed. More importantly, we can look at other parts of C++ to avoid
introducing a completely new syntax and instead, be consistent with existing bits (that we can't change, standardization is, for the most part, an additive only process).

## 3 - Consistent

> If you talk to every individual member of the standardization committee, and you said "Are you concerned about inconsistencies, and simplicity and ease of explanation?", they would all say "yes, those are very important".
> And they would not be lying. Every member of the committee would say yes, those are very important to me. But in my opinion, if you look at the result of their work, the resulting standardization document; The decisions they make ultimately,
as a committee do not reflect these concerns. - Scott Meyers

What do we mean about consistency?

We probably want template parameters lists to look somewhat like functions parameter lists.
Or maybe we want functions and lambda to look as much like one another as possible?
Or maybe should parameters declaration match variable declaration?
Should NTNTTP declaration and Type template parameter look alike somehow?
What should be done with auto and its multiple meanings?

There is mainly 2 kind of consistencies.
The first one is familiar sequences of tokens, syntactic patterns used in similar context through the language.
Of course, we can argue whether two contexts are similar enough to use the same contexts.
A familiar syntax used for a different purpose in a different context is inconsistent indeed.


But, I have found that consistency is fore-and-foremost, a good story.
In this case, consistency comes more from a mental model a developer has rather than from the syntax.

The heated `const Foo x` vs `Foo const x` is a recent demonstration of that (westconstia forever).
What you find consistent and intuitive in this context depends on the mental model you prefer. The same thing goes for details like `Foo* bar` vs `Foo *bar`.

Having a "consistency story" is akin to having a rationale on a proposal or imagining yourself teaching that syntax.
How do concepts fit into your mental model?

Syntax is just syntax but it might affect the way you think about the language.

At least, we probably can agree that we don't want to introduce a syntax so illogical and alien that it is inconsistent with everything else.


## 4 - Terse

Some people want the syntax to be as terse as possible and they really have nothing else to say about that.

But can terse be too terse?
Does verbosity hinder people 's ability to read code (reading code is much more frequent than writing it)?
Should we count individuals characters? Should symbols count double?
Does Perl have concepts?

## 5 - Verbose

Some people like syntax so much, Bjarne call them the "syntax people". We know little of the syntax people, where they come from or what their motivations are.
Like Ent, they don't write any C++ expression unless it takes a very large amount of exotic tokens to do so.
For them, any single template declaration should be preceded by "Hail to the Chief" and every single instantiation be as ceremonious as humanly possible.

The syntax people where first encountered in the 90s when C++ was being standardized.
At the time, templates and generic programming was rather novel, and people tend to be afraid of novel things.
And so, people were very keen to have a syntax for generic programming that served as a caution sign that they were indeed using templates.

Bjarne noticed that people tend to like new features to be verbose but often ask for a more terse syntax as they get more familiar with the feature.
Isn't that the definition of FUD?

Of course, the case can be made than generic programming may lead to increase code size which still isn't acceptable in the most constrained environments,
However, most embedded developers seem to not be aware that identifiers can be more than four letters long so I'm not sure that's the culprit.

What is certain though, is that it will be difficult to reconcile the idea that generic programming should be ceremonious
and that generic programming should be no different than non-generic programming.

And once again, "verbosity" is a bit subjective. What one considers verbose enough varies greatly.

## 6 - Forwarding references

We are finally getting to an actual technical concern.

`Foo &&` deduces a different type whether `Foo` is a type or the name of a template parameter.
In the first case, it's an r-value reference, in the second case it's a forwarding reference,
which might be a reference to an rvalue or a reference to an l-value with any cv-qualifier it might have.

[N4164](https://wg21.link/N4164), the paper that gave forwarding references their name, makes a great job of explaining what they are.
You might notice that "Forwarding references" have a name only since C++17, while they were introduced in C++11.

Forwarding references are an artifact of references collapsing and special rules for template argument deductions,
a topic notably covered by [Scott Meyers](https://www.youtube.com/watch?v=wQxj20X-tIU).
So while it took them a while to be named, forwarding references have always been fairly well understood.

But, it is not possible to distinguish forwarding references from r-value references without knowing the nature
of the entity they decorate as they share the same syntax. It's unclear if that was intentional at the time or whether it was seen as a neat trick,
but a lot of experts now believe it was a mistake to not introduce a different syntax for forwarding references.

As we look to introduce a short syntax, how can we distinguish forwarding references for r-value references?
That is, how can we distinguish concrete types from template parameters and concept names?

There are a few options

* Make sure parameters whose type is a template/concept-name are visually distinguished.
* Retroactively remove the ambiguity from the language. Some people have suggested `&&&` as a syntax to mean forwarding reference.
But of course, that ship has sailed, so even if we introduce a new unambiguous syntax, what to do of the old one?
* Choose to turn a blind eye on this issue.


## 7 - Non-type, Non-Template Template Parameters and Value Concepts

A template parameter can be a type, or a value.
Furthermore, concepts can either constrained a type or a value.
However, a given concept can never constrain both a type and a value - Even if constraining a value implicitly constrains its type.
For example, an "Even" concept that would check that `v % 2 == 0` can be applied to an `int` but not to a string or a double as neither
of those types has a `%` operator.

It seems a common misunderstanding that template value parameter (NTNTTP) can be mutated.
Then it would be legitimate to wonder whether a constraint should apply over the lifetime of said variable.
But in fact, as per the standard,

> A non-type non-reference template-parameter is a prvalue.
> It shall not be assigned to or in any other way have its value changed.
> A non-type non-reference template-parameter cannot have its address taken.

So a concept or set of constraints can only ever be applied at the time of instantiation.

The following snippet is valid, at no point can a concept constrains a runtime value.
That's what contracts are for!


```cpp
template <Even e> decltype(e) f()  {
    return e + 1;
}
[[assert: f<0>() == 1]];
f<1>(); // ill-formed
```

I don't think this is an actual issue people struggle with? If you think it's confusing, let me know!

To sum things up

* A template parameter can be a type or a value
* In a function signature, only types can be constrained
* We may want to constrain NTNTTP on both their value and their type.
* Types are significantly more common than NTNTTP in template definitions but in C++20
a lot more types can be used as template parameters so that might change slightly.

## 8 - Pleasing

Last and maybe least, if there is such thing as elegant code, maybe can we find a syntax that is not too unpleasant to our obsessives minds.
After all, the world is watching.

# Making sense of an ocean of proposals



































