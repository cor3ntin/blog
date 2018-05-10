---
title: "C++ Attributes"
date: 2018-04-22
---

In C++11, attributes were added as a way to standardized features such
as gnu `__attribute__`and msvc’s `__declspec`.

The language provisions for standard attributes as well as non-standard
attributes through the use of namespaces, though the behavior of
non-standard attributes was only settled for C++17. And sadly, as of
2018, neither GCC nor MSVC offer their vendor-specific attributes though
the portable C++ standard syntax.

Most standard attributes were added in C++14 and 17. You can find a list
on
[cppreference](http://en.cppreference.com/w/cpp/language/attributes).

At some point before C++11 came out, the C++ Standard draft defined the
`[[override]]` and `[[final]]` attributes. These features
were later converted to contextual keywords `override` and `final`. The
original proposal even suggested that default and deleted methods could
be made possible with the use of attributes. We now know these features
as `=default` and `=delete`.

And ever since, when adding a language feature to C++, the question of
whether that feature would be better served by a keyword or an attribute
needs to be asked and answered.

It recently came up for the now accepted `[[no_unique_address]]`
and the currently discussed `[[move_relocates]]` (which would, to the
extent of my understanding, be akin to a destructive move, a nice
performance optimization in some cases).

Systematically, a committee member will point out that “*A compiler
should be free to ignore attributes*”. One other will add “*Well,
actually, an attribute should not change the semantic of a program*”. Or
more accurately,

> Compiling a valid program with all instances of a particular attribute
> ignored must result in a correct implementation of the original
> program

*source :*[*p0840r0*](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0840r0.html)

This is sort of an unwritten rule, it’s not actually anywhere in the
standard, though I found this quote in the initial attribute proposal:

> What makes this a good candidate for attributes is that code that runs
> with these attributes also runs identically if the attributes are
> ignored, albeit with less type checking.

It makes a lot of sense in the case of non-standard attributes. We
should let the dark past of not-portable code and vendor-extensions
behind us. So having non-standard attributes being simply skipped over
is very useful and desirable indeed.

But what about standard attributes ? The compilers are free to ignore
those too.

So let’s say you use a `[[fallthrough]]`
attribute and have a 0-warning policy (say you compile with -WError),
the compiler is free to emit a warning and to fail the build.

In the case of `[[no_unique_address]]` ,
`sizeof` may return a different result
depending on whether the attribute is ignored or not, thereby letting
you affecting the semantic of the program. Which both show that the
committee doesn’t necessarily follow their own rules, but most
importantly that attribute being ignorable do not match the developer’s
intent.

Even as people learned to expect that vendors often implement the
standard in partial and opinionated ways, they probably don’t sprinkle
attributes just for kicks, or to give their code some artsy Christmas
ornaments. If someone goes to the trouble of marking a function
`[[nodiscard]]`, they probably really
want the result of a function to be checked. Maybe not checking the
return value could, for some reason, lead to a critical failure. Rockets
blowing up, patients dying.

There is, for existing attributes, no implementability concern either.
They all can be implemented on all hardware as they don’t impose
hardware requirement. I suppose there are some hardware for which
`[[caries_dependencies]]` makes no sense,
but on such hardware, `std::memory_order`
wouldn’t make sense either, making the point moot.

I’ve tried to ask several committee members the nagging question :
**Why ? Why attributes need to have no semantic meaning ?**

The answer I got was : *Because.*


{{< figure src="https://cdn-images-1.medium.com/max/800/1*arziEb8HG5FkhINW70umHw.gif" title="It is so" >}}

And it was hard for me to find more rationale behind that. And there is
nothing quite irksome as a rule without rationale.

A reason that I was given is that the committee might, in absence of
strong guidelines, use attributes to shovel more keywords into the
language, since adding attributes is easier than adding keywords:
introducing keywords might break someone’s code and understandably calls
for a stronger motivation. And letting attributes be everything might
make the language an ungodly soup of attributes.

That is certainly a valid concern. However, does the committee really
needs to impose rules onto itself ? They still need to vote on each and
every standard attribute that goes into the standard, and need to study
the relevance of the attribute with or without the existence of this
weird unwritten rule.

I don’t think there is a good general answer to whether attributes
should be keyword, or whether they should be imbued with semantic
meaning or not.

Instead, that question needs to be answered on a per-attribute basis.
Should `alignas` be a keyword ? Maybe not. Should `[[fallthrough]]`be one ?
Probably; that’s a good example of shoving a keyword as an attribute to
avoid breaking user’s code.

Ultimately, that sort of consideration is highly subjective, but I doubt
introducing seemingly arbitrary rules makes the design process any
easier, probably quite the opposite.

Instead, each proposed standard attribute (or keyword) needs to be
studied on its own merits, and maybe we will find attributes for which
it makes sense that they could be ignored — Which I don’t think is the
case for any of the existing attributes.

This might sound like bickering, and to some extent, it is. However it
might matter in the context of reflection.

#### **Reflection on attributes**

I think reflection on attributes may be the most important aspect of
reflection. Let say you want to serialize a class, and using reflection
to visit all the members, you may need to filter out some members that
you don’t want to serialize. One way to do that would be to (ab)use the
type system but a better system would probably be to use attributes to
tag the relevant members. That could open the door to amazing
serialization, RCP and ORM libraries (…even though you probably should
not use an ORM !)

Most people appear to see the value of such feature, however some argue
that it would be better to add another syntax, that could be called
*decorator.* It would essentially be the same thing as attributes, used
in the same place as attributes, but with a new syntax that would be
exempt of the “attribute should be ignorable” debate.

To me this really does not make sense. First, if a user chooses to
impart semantic meaning to attributes by the mean of reflection, then
it’s the user’s choice rather than the compiler’s, so portability
concerns do not apply. And of course, this feature may and would be used
to develop framework-specific behaviors and idioms, which some people
seem to be rather strongly against. But that is a need that exists and
is emulated today by the means of complicated macros or code generation,
often both (Qt).

Adding a new syntax, giving it another name over some philosophical
subtlety that will be loss on non-expert, would almost certainly add no
value to the language. And would of course add delay while that new
syntax is argued about. C++ is running out of tokens.

For example, which of the following do you think is the most readable ?

```cpp
[[deprecated]] QSignal<void> f();
[[deprecated]] @@qt::signal@@ void f();
[[deprecated]] [[qt::signal]] void f();
```

There are other challenges with reflecting on attributes. Attributes can
take parameters that are presently defined as being a token soup. Hem, I
mean token-sequence sorry. A well balanced token-soup-sequence. Which I
guess makes total sense for tooling and compilers. But is pretty
pointless when it comes to reflection.

I think there are solutions to this problem, namely that
reflectable-attributes be declared in the source before use and allow
literals and `constexptr`as their
parameters, rather than arbitrary tokens.

I’ve put more details on that on GitHub. Note that this is not a
proposal and I don’t intend to make one, at least not while the
“attribute should be ignorable” mantra exists.


[cor3ntin/CPPProposals](< ref "https://github.com/cor3ntin/CPPProposals/blob/master/attributes/attributes.rst" >)


In the end, attributes could have a variety of uses:

 - Compiler directives for optimizations and diagnostics

 - Instructions and meta data for external tooling

 - Filtering and decoration by the use of both reflection of code
injection (code injection, would let people use attributes as full
blown decorators similar in expressiveness to the feature of the
same name offered by Python)

 - Contracts ( `[[expects:]]`and `[[ensures:]]`) ; though these
syntax are different enough that i’m not sure they still qualify as
attributes.

But as things stand, I feel like attributes are criminally underused and
crippled.

What attributes would you like to see ?
