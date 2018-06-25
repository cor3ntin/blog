---
title: "The tightly-constrained design space of convenient syntaxes for generic programming"
date: 2018-06-13T11:32:09+02:00
---

<div style="font-size: larger; padding:1em; margin-top:2em; margin-bottom:2em; background:#FFF2DB; color:#666">

<p> Please take the quick survey on concept syntax at the <a href="#survey"> end of this article</a>.
</p>

</div>

Did you know that the Concept TS was merged into the Working Draft in July 2017, in Toronto?
And we are a planck length away from merging the Range TS in C++20 as well, including a few goodies such as projections,
contiguous ranges/iterators and ranges adaptors?
We also added a bunch of general-purpose concepts in the `std` namespace in Rapperswil.

Concepts have been 3 decades in the making and the Ranges TS is a huge body of work.
Yet, I feel like a lot of people are unaware of these great features that are coming to a compiler near them.

It might be that only GCC has an implementation of concepts (that does not quite match the TS and that is easily tripped), making experimentation a bit hard.
Or maybe people are tired of waiting? After all, we were promised ~~jetpacks~~ concepts in C++11, C++14, C++17.

Or maybe the expert-friendly syntax of concept usage scares people away?

# What are concepts?

Truth is, there is little concepts can do that can not already be achieved with C++17 and (a lot of) SFINAE.
Eric Niebler's widely popular `range-v3`, which was the foundation of the Ranges TS makes heavy use of ""concepts"" using a lot of
[SFINAE tricks and a few macros](http://ericniebler.com/2013/11/23/concept-checking-in-c11/). And honestly, using `range-v3` to define or
refine your own concepts is rather easy.
Still, without a lot of metaprogramming tricks that most developers should not be expected to fully understand, SFINAE is tricky and error-prone.
Concepts aim to provide an easy way to describe complex requirements on individual types and advanced overload sets.

The second thing that concepts offer is better error messages (even if this is, of course, a quality-of-implementation matter).
The compiler can pinpoint exactly what requirement(s) a type is missing for a given template instantiation but it can't
know which template you were trying to instantiate as it can't read your mind to resolve ambiguities. [Yet](https://www.aaai.org/ocs/index.php/AAAI/AAAI17/paper/view/14603).

It's probably [less magical than what you might expect](https://godbolt.org/g/g8gfLM), so it will not absolve
C++ developers from understanding cryptic compilation errors spawned somewhere inside a deep template instantiation stack,
however to an experienced developer, the errors will be much more explicit.

So, it does not seem too inaccurate to view Concepts as sugar coating over SFINAE, with the added bonus of more explicit errors messages.
It may not seem very exciting, admittedly.

But since Alexander Stepanov and Bjarne Stroustrup dreamed concepts a few things happened.
First, meta-programming tricks, knowledge, and libraries, as well as more robust compiler implementations enabled libraries such as `range-v3` to exist.
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
How to distinguish generic functions from the non-generic ones? What about collapsing ~~universal~~ forwarding references?

As the ink was flowing, C++14 shipped. As C++17 sailed, defenders and detractors of the concept abbreviated syntax dug trenches until progress on the Concept TS
came to a gloomy stop.

In this context, a brave soul suggested that we could maybe remove the abbreviated syntaxes from the TS and merge the non-controversial bits in the TS.
And for a little while, a truth was established, allowing concepts to be merged in the Working Draft, while Tom Honermann enjoyed his well deserved moment of glory.

**However**.

The committee still wanted a ~~short~~ ~~abbreviated~~ ~~terse~~ ~~natural~~ convenient syntax. They just could not agree on which syntax was best. It was back to the drawing board.


You might think that getting consensus on syntax would be easier.
It turns out the design space is ridiculously complicated, so let me try to describe some of the numerous constraints.


# The Design Space

## 0 - The meaning of void f(ConceptName a, ConceptName b)

Up until last year, some people argued that given `void f(ConceptName a, ConceptName b)`, `a` and `b` should resolve to the same type.

This is a quote from the [original Concept-Lite proposal](https://wg21.link/n3701):

> What if we need two argument types of the same concept? Consider <br/>
> `void sort(Random_access_iterator p, Random_access_iterator q);` <br/>
> For this to make sense, `p` and `q` must be of the same (random-access iterator)
> type, and that is the rule. By default, if you use the same constrained parameter
> type name for two arguments, the types of those arguments must be the same.
> We chose to make repeated use of a constrained parameter type name imply
> “same type” because that (in most environments) is the most common case,
> it would be odd to have an identifier used twice in a scope have two different
> meanings, and the aim here is to optimize for terse notation of the simplest case.


Fortunately, [this issue was resolved](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0464r2.html),
and there is now a tacit(?) consensus that each parameter should be deducted separately and be of potentially different types.

So, in fact, _some_ progress was made on the convenient syntax and things are moving in the right direction


## 1 - It's purely a syntax issue.

* Concepts are in the Working Draft.
* Every imaginable set of constraints can be applied to type and methods using the non-convenient syntax as per the Working Draft.
* The compiler (and by extension, tools) needs no syntax whatsoever to distinguish concepts, types, values, type-concepts, value-concepts.
* There may be some questions regarding references but the same solution can be applied regardless of the actually chosen syntax
* The question then is what the best way to please the finicky human developers might be.

## 2 - Simple and natural

The main goal being making templates simpler for most people, we need to find a simple syntax.
Something easy to teach and easy to learn.
Hopefully intuitive. But people's intuitions change.
People have different expectations depending on their background, the other languages they know, their skill level with C++...
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

We probably want template parameter lists to look somewhat like function parameter lists.
Or maybe we want functions and lambda to look as much as one another as possible?
Should parameters declaration match variable declaration?
Should <acronym title="Non-template, Non-type, template parameter">NTNTTP</acronym> declarations and type template parameters look alike somehow?
What should be done with auto and its multiple meanings?

There is mainly 2 kind of consistencies.
The first one is familiar sequences of tokens, syntactic patterns used in similar contexts through the language.
Of course, we can argue whether two contexts are similar enough to use the same syntax.
A familiar syntax used for a different purpose in a different context is inconsistent indeed.


But, I have found that consistency is, first and foremost, a good story.
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

Some people like syntax so much, Bjarne calls them the "syntax people". We know little of the syntax people, where they come from or what their motivations are.
Like Ent, they don't write any C++ expression unless it takes a very large amount of exotic tokens to do so.
For them, any single template declaration should be preceded by "Hail to the Chief" and every single instantiation be as ceremonious as humanly possible.

The syntax people where first encountered in the 90s when C++ was being standardized.
At the time, templates and generic programming were rather novel, and people tend to be afraid of novel things.
And so, people were very keen to have a syntax for generic programming that served as a caution sign that they were indeed using templates.

Bjarne noticed that people tend to like new features to be verbose but often ask for a more terse syntax as they get more familiar with the feature.
Isn't that the definition of FUD?

Of course, the case can be made that generic programming may lead to increased code size, which still isn't acceptable in the most constrained environments.

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

As we look to introduce a short syntax, how can we distinguish forwarding references from r-value references?
That is, how can we distinguish concrete types from template parameters and concept names?

There are a few options

* Make sure parameters whose type is a template/concept-name are visually distinguished.
* Retroactively remove the ambiguity from the language. Some people have suggested `&&&` as a syntax to mean forwarding reference.
But of course, that ship has sailed, so even if we introduce a new unambiguous syntax, what should be done with the old one?
* Choose to turn a blind eye on this issue.


## 7 - Non-type, Non-Template Template Parameters and Value Concepts

A template parameter can be a type, or a value (or a template).
Furthermore, concepts can either constrain a type or a value.
However, a given concept can never constrain both a type and a value - even if constraining a value implicitly constrains its type.
For example, an "Even" concept that would check that `v % 2 == 0` can be applied to an `int` but not to a string or a double as neither
of those types has a `%` operator.

It seems a common misunderstanding that template value parameter (<acronym title="Non-template, Non-type, template parameter">NTNTTP</acronym>) can be mutated.
Then it would be legitimate to wonder whether a constraint should apply over the lifetime of said variable.
But in fact, as per the standard,

> A non-type non-reference template-parameter is a prvalue.
> It shall not be assigned to or in any other way have its value changed.
> A non-type non-reference template-parameter cannot have its address taken.

So a concept or set of constraints can only ever be applied at the time of instantiation.

The following snippet is valid; at no point can a concept constrain a runtime value.
That's what contracts are for!


```cpp
template <Even e> decltype(e) f()  {
    return e + 1;
}
[[assert: f<0>() == 1]];
f<1>(); // ill-formed
```

I don't think this is an actual issue people struggle with? If you think it's confusing, let me know!

<em> Note : Concepts may be accompanied by runtime requirements not expressed as code but that the function expects to hold true.
They may take the form of comment or be implicitly assumed by developers. </em>

To sum things up:

* A template parameter can be a type or a value
* In a function signature, only types can be constrained
* We may want to constrain <acronym title="Non-template, Non-type, template parameter">NTNTTP</acronym> on both their value and their type.
* Types are significantly more common than <acronym title="Non-template, Non-type, template parameter">NTNTTP</acronym> in template definitions but in C++20
a lot more types can be used as template parameters so that might change slightly.

## 8 - Pleasing

Last and maybe least, if there is such thing as elegant code, maybe can we find a syntax that is not too unpleasant to our obsessives minds.
After all, the world is watching.

# Making sense of an ocean of proposals

## Overview

A tony table is worth a thousand words

<div style='width: 80vw; position: relative; left: 50%; right: 50%;margin-left: -40vw; margin-right: -40vw;'>

<div style="margin:auto;overflow-x:auto;">
<table>
<tr>
    <th> <a href="http://eel.is/c++draft/temp.param#nt:constrained-parameter">C++20 draft</a>
    </th>
    <th> <a href="https://wg21.link/n3701">Concept Lite</a> </th>
    <th> <a href="https://wg21.link/p1079">Bjarne's "minimal solution"</a> </th>
    <th> <a href="https://wg21.link/p0745r1">In Place Concept</a> </th>
    <th> Adjective <sup><a href="#fn1" id="ref1">1</a></sup>
        <a href="https://wg21.link/p0807">P0807</a>
        <a href="https://wg21.link/p0791">P0791</a>
        <a href="https://wg21.link/p0873">P0873</a>
    </th>
</tr>
<tr>
    <th colspan="5" style="text-align:center"> Simple function </th>
</tr>
<tr>
<td>
    <pre>template &lt;Container C>
void sort(C & c);</pre>
</td>
<td>
    <pre>void sort(Container &c);</pre>
</td>
<td>
    <pre>template void sort(Container &c);</pre>
</td>
<td>
    <pre>void sort(Container{} &c);</pre>
</td>
<td>
    <pre>void sort(Container auto &c);</pre>
</td>
</tr>

<tr>
    <th colspan="5" style="text-align:center"> Function with type name introduction </th>
</tr>
<tr>
<td>
    <pre>template &lt;Container C>
void sort(C & c);</pre>
</td>
<td>
    <pre>Container{C} void sort(C &c);</pre>
</td>
<td>
    <pre>template &lt;Container C> void sort(C &c);</pre>
</td>
<td>
    <pre>Container{C} void sort(C &c);</pre>
</td>
<td>
    <pre>template &lt;Container C>
    void sort(C &c);</pre>
</td>
</tr>

<tr>
    <th colspan="5" style="text-align:center"> Lambdas </th>
</tr>
<tr>
<td>
    <pre>[]&lt;Container C>(C & c) {};</pre>
</td>
<td>
    <pre>[](Container &c){};</pre>
</td>
<td>
    <pre>[](Container & c) {};</pre>
    <pre>[]&lt;Container C>(C & c) {};</pre>
</td>
<td>
    <pre>[](Container{} &c){};</pre>
    <pre>[]&lt;Container{C}>(C &c){};</pre>
</td>
<td>
    <pre>[](Container auto & c) {};</pre>
    <pre>[]&lt;Container C>(C & c) {};</pre>
</td>
</tr>

<tr>
    <th colspan="5" style="text-align:center"> Template parameter list </th>
</tr>
<tr>
<td>
    <pre>template&lt;Container C></pre>
</td>
<td>
    <pre>template&lt;Container C></pre>
</td>
<td>
    <pre>template&lt;Container C></pre>
</td>
<td>
    <pre>template&lt;Container{C}></pre>
</td>
<td>
    <pre>template&lt;Container C></pre>
    <pre>template&lt;Container typename C></pre>
</td>
</tr>

<tr>
    <th colspan="5" style="text-align:center"> Non type, Non-template template parameter constrained on type</th>
</tr>
<tr>
<td>
    <pre>template&lt;auto N>
    requires Unsigned&lt;decltype(N)></pre>
</td>
<td>
    🚫 Same syntax as the working draft
</td>
<td>
    <pre>template&lt;Unsigned_value N></pre>
</td>
<td>
    <pre>template&lt;Unsigned{Type} N></pre>
</td>
<td>
    <pre>template&lt;Unsigned auto N>
</td>
</tr>


<tr>
    <th colspan="5" style="text-align:center"> Non type, Non-template template parameter constrained on value</th>
</tr>
<tr>
<td>
    <pre>template&lt;auto N>
    requires Even&lt;decltype(N)></pre>
</td>
<td>
    🚫 Same syntax as the working draft
</td>
<td>
    <pre>template&lt;Even N></pre>
</td>
<td>
    🚫 Same syntax as the working draft
</td>
<td>
    <pre>template&lt;Even auto N>
</td>
</tr>

<tr>
    <th colspan="5" style="text-align:center"> Non type, Non-template template parameter constrained on both value and type</th>
</tr>
<tr>
<td>
    <pre>template&lt;auto N>
    requires Unsigned&lt;decltype(N)>
        && Even&lt;N> </pre>
</td>
<td>
    🚫 Same syntax as the working draft
</td>
<td>
    🚫 Same syntax as the working draft
</td>
<td>
    🚫 Same syntax as the working draft
</td>
<td>
    <pre>template&lt;Unsigned Even auto N>
</td>
</tr>


<tr>
    <th colspan="5" style="text-align:center"> Dependent types </th>
</tr>
<tr>
<td>
    <pre>template&lt;typename A, typename B>
    requires Swappable&lt;A, B>
    void foo(A & a, B & b);</pre>
</td>
<td>
    <pre>Swappable{A, B} void foo(A & a, B & b);<pre>
</td>
<td>
    <pre>template&lt;Swappable{A, B}>
void foo(A & a, B & b);</pre>
</td>
<td>
    <pre>template&lt;Swappable{A, B}>
    void foo(A & a, B & b);</pre>
    <pre>void foo(Swappable{A,B} & a, B & b);</pre>
</td>
<td>
    🚫 Same syntax as the working draft
</td>
</tr>

<tr>
    <th colspan="5" style="text-align:center"> Identical types  <sup><a href="#fn2" id="ref2">2</a></sup> </th>
</tr>
<tr>
<td>
    <pre>template&lt;Container A>
void foo(A & a, A & b);</pre>
</td>
<td>
    <pre>void foo(Container & a, Container & b);</pre>
</td>
<td>
    🚫 Same syntax as the working draft
</td>
<td>
    <pre>void foo(Container{A} & a, A & b);</pre>
    <pre>void foo(Container{A} & x, C{A} & b);</pre>
</td>
<td>
    🚫 Same syntax as the working draft
</td>
</tr>


<tr>
    <th colspan="5" style="text-align:center"> Identical constraints on different types  <sup><a href="#fn2" id="ref2">2</a></sup> </th>
</tr>
<tr>
<td>
    <pre>template&lt;Container A, Container B>
void foo(A & a, B & b);</pre>
</td>
<td>
    🚫 Same syntax as the working draft
</td>
<td>
    <pre>template
void foo(Container & a, Container & b);</pre>
</td>
<td>
    <pre>void foo(Container{A} & a, Container{B} & b);</pre>
</td>
<td>
    <pre>void foo(Container auto & a, Container auto & b);</pre>
</td>
</tr>


<tr>
    <th colspan="5" style="text-align:center"> Unconstrained type </th>
</tr>
<tr>
<td>
    <pre>template&lt;typename Foo>
void foo(Foo & a);</pre>
</td>
<td>
    <pre>void foo(auto & a);</pre>
</td>
<td>
    <pre>void foo(auto & a);</pre>
</td>
<td>
    <pre>void foo(auto & a);</pre>
</td>
<td>
    <pre>void foo(auto & a);</pre>
</td>
</tr>


<tr>
    <th colspan="5" style="text-align:center"> Multiple constraints <sup><a href="#fn3" id="ref3">3</a></sup></th>
</tr>
<tr>
<td>
    <pre>template&lt;typename Foo>
    requires Container&lt;Foo>
        && Iterable&lt;Foo>
void foo(Foo & a);</pre>
</td>
<td>
    🚫 Not proposed
</td>
<td>
    🚫 Not proposed
</td>
<td>
    🚫
</td>
<td>
    <pre>void
foo(Iterable Container auto & a);<pre>
</td>
</tr>


<tr>
    <th colspan="5" style="text-align:center"> Return value</th>
</tr>
<tr>
<td>
    <pre>template &lt;Container C>C foo();</pre>
    <pre>template &lt;Container C>auto foo() ->C;</pre>
</td>
<td>
    <pre>Container foo();</pre>
    <pre>auto foo() -> Container;</pre>
</td>
<td>
    <pre>Container foo();</pre>
    <pre>auto foo() -> Container;</pre>
</td>
<td>
    <pre>Container{} foo();</pre>
    <pre>auto foo() -> Container{};</pre>
</td>
<td>
    <pre>Container auto foo();<sup><a href="#fn4" id="ref4">4</a></sup></pre>
    <pre>auto foo() -> Container auto;<sup><a href="#fn4" id="ref4">4</a></sup></pre>
    <pre>auto foo() -> Container;</pre>
</td>
</tr>



<tr>
    <th colspan="5" style="text-align:center"> Local variables type checking</th>
</tr>
<tr>
<td>
    <pre>auto c = get_container();
static_assert&lt;Container&lt;decltype(c)>()>;</pre>
</td>
<td>
    🚫 Not proposed
</td>
<td>
    🚫 Not proposed
</td>
<td>
    <pre>Container{} c = get_container();<pre>
</td>
<td>
    <pre>Container auto c = get_container();<pre>
</td>
</tr>


<tr>
    <th colspan="5" style="text-align:center"> Visual distinction of template function</th>
</tr>
<tr>
<td>
    &#x2714;
</td>
<td>
    🚫
</td>
<td>
    &#x2714;
</td>
<td>
    &#x2714;
</td>
<td>
    &#x2714;
</td>
</tr>

<tr>
    <th colspan="5" style="text-align:center"> Visual distinction of concepts and concret types</th>
</tr>
<tr>
<td>
    &#x2714;
</td>
<td>
    🚫
</td>
<td>
    🚫
</td>
<td>
    &#x2714;
</td>
<td>
    &#x2714;
</td>
</tr>

<tr>
    <th colspan="5" style="text-align:center"> Visual distinction of value-concepts and type-concepts</th>
</tr>
<tr>
<td>
    &#x2714;
</td>
<td>
    🚫
</td>
<td>
    🚫
</td>
<td>
    Yes, but confusing
</td>
<td>
    &#x2714;
</td>
</tr>
</table>

<span id="fn1"><a href="#ref1" title="Jump back">1</a>. There have been several adjective syntax papers (I'm the author of one of them).
This table attempt to aggregate them.</span>
<br/>
<span id="fn2"><a href="#ref2" title="Jump back">2</a>.
There are other ways to achieve the same result, including using <code>decltype</code> but these solutions do not behave the same way in an overload set.</span>
<br/>
<span id="fn3"><a href="#ref3" title="Jump back">3</a>. Proposed as a possible extension.</span>
<br/>
<span id="fn4"><a href="#ref4" title="Jump back">4</a>. In a return value, <code>auto</code> could be optional (no forwarding reference issue) and is offered as a way to maintain a coherent consistency story.</span>

</div>
</div>

## Bjarne "simple" proposal

<p><a href="https://wg21.link/p1079">P1079</a></p>

I think Bjarne's proposal is probably what the concept syntax should have been if it has been designed before C++.
It's simple, minimalist and therefore easy to use and teach.
The `template` keyword was added to please the syntax people and allow distinguishing between functions and functions templates.

However, this proposal chooses to ignore the rvalue/forwarding reference ambiguity issue.
Indeed, the `template` keyword tells you nothing about the nature of each individual parameter type.

The author believes that the rvalue/forwarding reference ambiguity should be fixed rather than adapting the syntax around that issue.
While this would indeed be great, all the committee members I talked to think this issue can not be fixed in any meaningful way.
That ship has sailed when C++ shipped.

Interestingly, it allows a shorthand syntax inspired by concept-lite to declare multiple types with dependent constraints.
On the other hand, it makes working with <acronym title="Non-template, Non-type, template parameter">NTNTTP</acronym> a bit clumsy and ambiguous.

## Herb's "In-Place" proposal

<p><a href="https://wg21.link/p0745">P0745</a></p>

Inspired by the notion of "concept introducers" that were initially in the TS,
this syntax manages to be both the most expressive, and the tersest.
This means that you can declare and constrain the more complicated functions of the STL in a single line.
It makes working with constraints involving multiple types or having parameters with identical types really easy.
It also makes possible to visually distinguish concepts from concrete types

But, in order to do do that, a few sacrifices are made

 * `template<Unsigned{N}>` declares `N` to be a type while `Unsigned{} N` is a value - whose type is unsigned.
While this is somewhat logical, I don't think it will be obvious to beginners.
 * It is not possible to constrain a value with a value-concept
 * The syntax is... novel. In the simple case (aka `void sort(Sortable{} & c);`),
the syntax will not be familiar to C++ developers or people coming from another language.

I also dislike that it introduces dependencies between separate declarations:
Take <br/>
    `void f(C{A} _1, A _2)`:
In this example, the declaration of `_2` depends on the declaration of `_1`.
Of course, this is achievable already with `decltype`, but introducing a core syntax will make this pattern more widespread and
it makes refactoring and tooling harder.


## Adjective syntax.

<p><a href="https://wg21.link/p0807">P0807</a>, <a href="https://wg21.link/p0791">P0791</a>, <a href="https://wg21.link/p0873">P0873</a></p>

Take any existing variable, generic function/lambda parameter. Stick a Concept name on the left. This entity is now constrained.
Existing syntax is not modified (Concepts names are added to the left).
To make things terser, we make `typename` optional in a template parameter declaration.
This is the adjective syntax in a nutshell.

Concepts are distinguished from types by the presence of `auto` - `auto` is a familiar keyword meaning "deduce the type".
So it's easy to distinguish template functions from non-template functions.

The adjective syntax also offers a natural model to work with <acronym title="Non-template, Non-type, template parameter">NTNTTP</acronym> parameters.

This syntax focus on simplicity, and consistency while making sure that types and concepts are distinguished in order not to introduce
more trap into the language.

But because it focuses on making the simple case simple, it is a bit more verbose than other proposed syntax and a `require` clause is necessary
to specify constraints on multiple types.

# Conclusion

I tried to stay neutral, but it's no secret that I like the adjective syntax the best.
It has a good story for forwarding references, <acronym title="Non-template, Non-type, template parameter">NTNTTP</acronym> and is consistent with the rest of the language and easy to teach. It also has no traps or ambiguities.

But each proposed syntax has both merits and drawbacks so hopefully, the most popular one will prevail!
To this end, I set up a survey.

# Survey

This is very informal but it may influence me or others to write papers accordingly.
And I'm very curious to know what your opinions on that matter so please take that survey.
It's hopefully short enough, and if you have more to say, leave a comment!

On mobile you can access the survey [via this link](https://goo.gl/forms/RiHCk52yvfYKKge02).

I hope you enjoyed this article!

<iframe src="https://docs.google.com/forms/d/e/1FAIpQLSdWtc712Hp479exxo6j2aDcWjRrsxP2ORzpoxub2rMv5BB0vA/viewform?embedded=true" width="800" height="2000" frameborder="0" marginheight="0" marginwidth="0">
</iframe>




# Acknowledgments

Thanks to Simon Brand, Christopher Di Bella, Vinnie Falco, Tom Honermann, Martin Hořeňovský and Patrice Roy for proof-reading and offering valuable feedbacks.











