---
title: "Parenthesis"
date: 2019-02-26T07:00:36-10:00
draft: true
---

```cpp
foo();
```

This is `foo`. But what is `foo`?
An astute reader may assume `foo` is a function.

What kind of function?
It is obviously not a virtual function. But it might be a function pointer. It might be a function object.
It might even be a shiny post-modern lambda.
Or the constructor of some class `foo`.

It doesn't matter.
All these things amount to plain, old and boring functions.
You can call them. It creates a stack frame with its own scope and local variables, some computation happen, registers are shuffled around and
ultimately you may get a value representing the result of that runtime computation, aptly named return value.
This functions can also be parametrized with runtime values.

```cpp
int foo(int);
foo(42);
```

Parameters can be references, pointers, values. All of this is just boring register shuffling.

BUT

It might very well be that `foo()` is a macro that **expands** to something. *Anything*.
Maybe a hexadecimal encoding of a picture of a famous Swedish sunken ship. Or a string containing the whole content of Wikipedia.
It might expand to javascript, nothing, Perl.
If you are lucky, it might even expand to valid C++ but there is no guarantee of that.
Macros are powerful. They can have parameters too.

But they expand to what amount to a token soup. They are blissfully unaware of C++ syntactical and semantics rules.



So there you have it. `foo` is either a function or a macro.
Wait, is that it?

A clever compiler may decide to _inline_ `foo` (this has little to do with the `inline` keyword).
Inlining is a transformation such that the compiler will inject the body of the function directly inside callers as to avoid
creating a stack frame, which is expensive. This is purely a compiler optimization, unobservable under the as-if rule.

Yet, inlined functions expand as much as they are called. Can such a function be considered a macro?

An even more clever compiler can further decide to evaluate to function at runtime if it is able and not to lazy to do so.
This also falls under the _as-if_ rule. Now the compiler can get rid of both stack frame and runtime costs by _constant evaluating_ everything.

Then there are `constexpr` functions.
`constexpr` is merely a contract between a developer and the compiler that a function _can_ be and will remain constant evaluatable.
This contract which is part of the API (and the ABI for some reason), let you use `constexpr` functions in other `constexpr` contexts.

It doesn't mean that a given function at a given point in the program will always be constant-evaluated. Even if you might expect so, the compiler might get
lazy and complacent.

Which brings us to `consteval` functions or so-called immediate functions. These are functions that are always constant-evaluated. They have no address
so they can not be called at runtime.
A motivational example for `consteval` is reflection, where the information on types is only available at compile time.
`std::source_location` and `std::embed` are other examples of utilities making use of `consteval`.

It's getting complicating. But we are not done yet!

There are also template functions.
Which would not be that interesting and unusual if not for non-type  template parameter

```cpp
template<auto a>
void foo(int b);
```

such that `foo<1>(0);`  and `foo<2>(0);` call two different version of `foo`. `a` is always evaluated at compile time while `b` is evaluated at runtime.
of course, you can miss and match features.
For example


```cpp
template<auto a>
constexpr void foo(int b);
```


One non-existing feature often discussed is `constexpr` parameters.
You would write


```
void foo(constexpr int a);
```

such that `a` is evaluated at compile time. Note that, for historical reasons, `constexpr` variables are always _constant-evaluated_ and should probably be called `consteval` instead.



To be less confusing, less rewrite our function to:

```
void foo(consteval int a);
```

Which beg the question: what is the difference between these two declarations?
```
void foo(consteval int a);

template<auto a>
constexpr void foo();
```

Since `consteval` parameters are only an idea, we can only speculate.
But I guess there is a couple of ways to implement them.

The first would be to evaluate the parameter at the call site at compile type,
store it in read-only memory and push the address of the object resulting from
that compile-time evaluation onto the stack.

Aka, it would be equivalent to.

```cpp
void foo(const auto & a);
consteval int a = compiletime_computation();
foo(a);
```

The second option would be to make functions with `constexpr` parameters
function templates where each constexpr parameter becomes a non-type template parameter.
The second option might, in theory, be more efficient but would create quite a lot of template
bloat.

Anyway.


We have functions, functions evaluated at compile time, functions never evaluated at runtime, functions guaranteed to be evaluatable at compile time, functions whose definition is injected into the call site, function templates and functions whose parameters are evaluated at compile time.
And it-was-the sixties unscoped, unchecked wild parametrized token replacement. aka macros.

I am afraid that the list is not complete. Consider the following:

```cpp
void foo(int);
void foo(double);
```

Now, `foo` is no longer a function, but an overload set. An overload set then
is a set of 1 or more function. In other words, `foo` is both a bunch of functions
and a name.
Overload sets are currently poorly handled by C++.
//TODO LINKs


All of this leaves us with a wild and complicated taxonomy of callables and expandables entities in C++.
If it feels like it grew organically it is because it did. All of these features were added
over time, and the whole is not necessarily as cohesive as one would like.


Have you noticed that enormous gap, the giant evolutionary void between macros and immediate functions?

There are basically two schools of thoughts when it comes to preprocessor macros in C++.
People either think that they are sufficient or that nothing can replace their expressiveness
and so feel that the design space is not worth exploring.

Or that macros are so terrible that their name shall not be spoken and that the whole concept is cursed and should be left alone.
As if they Feared an Untamed Dragon reign over that landscape.


Yet, a few proposals have emerged in that design space.

 * [P0927] Towards A (Lazy) Forwarding Mechanism for C++
 * [P1221] Parametric Expressions



I do not find [P0927] very interesting.
It strives to allow evaluation of parameters in the context of the callee rather than the callee.
Lazy evaluation is useful in a lot of contexts and the paper gives a lot of good motivational use cases.
The issue is, a macro system gives you lazy evaluation and a whole more.
[P0927] basically suggest a bit of syntactical sugar over expressions-wrapped-in-a-lambda.
Nothing new under the sun.

I frankly hope this proposal will not be pursued, as it offers a narrow solution to a general problem,
which is something that has always been an issue in that design space.

# Parametric Expressions

Parametric Expressions, on the other hand, is a lot more exciting.
There are a lot of things to love about Jason Rice's work.

But, I have to complain about the name. _Parametric Expressions_.
Jason is very careful not to awaken the Fearsome Undefeated Deamon, but let's call a cat a cat, shall we?
What this paper proposes is a full-blown syntactical macro system for C++.

Oh no. I said the M word. Not this one. The other M word.

But macros, done right, are a good thing.
Repeat after me: "There is more than one kind of macros and macros are good for me".

We also need to get our terminology in order.
Macros don't get called or invoked, they are not callable.
No stack frame is involved, not even at a conceptual level
Instead, they get _expanded_.
The code is injected at the expansion site.
This is what makes macro fundamentally different from functions.
But you will notice that inline*d* functions and immediate functions can be thought of as
expanding.


I am unfair. _Parametric Expressions_ is not a completely terrible name. Indeed, they are entities
taking parameters and expanding to expressions.
Notably, they can not expand to statements, even less partially-formed statements. And mercifully, they
can never expand to an unchecked token soup.