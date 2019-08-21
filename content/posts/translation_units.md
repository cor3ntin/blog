---
title: "Translation units considered harmful ?"
date: 2018-10-29
---

Let say you have some struct `square` you want to compute the area of.

`struct square { int width; }`

You could of course do that:

`int area(square s) { return s.width * s.width; }`

But, your friend Tony told you to use more functions, so instead you do that
```cpp
int area(square s) { return width(s) * width(s); }
int width(square s) { return s.width; }
```

`area` being the function you really care about it is defined first - after all, code reads from top to bottom.\\
Or so argues [Clean Code](https://en.wikipedia.org/wiki/Robert_C._Martin).

As you may have guessed from the lack of `;` after the struct's closing bracket, the above code is written in D.
I figure my readership isn't really into D, so maybe you would prefer some **Rust**?

```rs
pub fn area(square: Square) -> i32 { return width(s) * width(s) }
pub fn width(square: Square) -> i32 { return s.width }
pub struct Square { width: i32 }
```

You can even compute the area of you square **_at scale_** with go

```go
func Area(s square) int { return width(s) * width(s); }
func width(s square) int { return s.width }
type square struct { width  int }
```
Or even **Swift**ly.

```swift
func area(s: Square) -> Int { return width(s:s) * width(s:s); }
func width(s: Square) -> Int { return s.width }
struct Square { var width:Int = 0; }
```

But of course, _you_ will worry about the overhead and will want the language the most performant (that's not a word).
Eager to please and impress, let me copy the D code and add that oh-so-important comma.

```cpp
struct square { int width; };
int area(square s)  { return width(s) * width(s); }
int width(square s) { return s.width; }
```

That's nice, isn't it?  Interesting how most languages look alike.
Hum, wait, that doesn't work???!!!

`error: 'width' was not declared in this scope`

But, you stupid thing, it's _RIGHT THERE_.\\
I declared everything in the global scope like a maniac, can't you see?

Alas, the standard makes the compiler blind.

> In the definition of a function that is a member of namespace N, a name used after the function's declarator-id23 shall be declared before its use in the block in which it is used or in one of its enclosing blocks ([stmt.block]) or shall be declared before its use in namespace N or, if N is a nested namespace, shall be declared before its use in one of N's enclosing namespaces.

Of course, this makes no sense, a compiler can really easily parse the declaration independently of the definition, as
proven by other languages. Or you know, C++ classes. (imagine replacing a big namespace with a class full of static methods and nested types)
Unless of course, it's a performance thing.
But, you are a very great engineer, so you wouldn't let a source file grow above a few hundred lines of code, would you?
I bet your code is beautiful, like this small self-contained super useful program

```cpp
#include <iostream>
int main () {
    std::cout << "Hello world\n";
}
```

Which on my system expands to about *33000* lines of code. The freaking thing. But more on that later.

Let's go back to square one.
C++, in its infinite wisdom, lets us forward-declare functions, so we can write this:

```cpp
struct square { int width; };
int width(const square& s);
int area(const square& s)  { return width(s) * width(s); }
int width(const square& s) { return s.width; }
```

Which is nice and dandy, if you squint.

Besides requiring you to get the exact declaration of functions perfectly right - which is hard to maintain, lots of entities are not forward-declarable,
notably type alias, templated types, etc.
Which is an odd limitation given that where forward declaring a function require you
to know the precise signature, for types you are merely trying to introduce a name.

## noexcept

You will notice that `area` never throws.
That is, there is no subexpression of `area` that can throw, ever.

You can check that it does not.

`static_assert(noexcept(area(square{})));`

Inevitably,  that fails.
`error: static assertion failed`.
We indeed forgot to tell the compiler that our function could not throw.

```cpp
int width(const square& s) noexcept;
int area(const square& s) noexcept { return width(s) * width(s); }
int width(const square& s) noexcept { return s.width; }
```

Notice that we need to add `noexcept` on all declarations, including the forward declarations.
And, you can lie to the compiler pretty easily.

```cpp
int area(const square& s) noexcept {
    return width(s) * width(s);
}

int width(const square& s) {
    throw 42;
}
```

The above code will `std::terminate()`, you know that the compiler knows that, everybody knows that.

So...what functions should be marked `noexcept`?
It's pretty simple actually. All the functions that can not throw.
That is the functions that:

* Don't contain a `throw` exception
* Don't call non-noexcept functions

Notice the double (triple?) negative.

So you, as a developer striving to mark all function that can be `noexcept` as such,
have to walk the call tree recursively until you can ascertain that the call chain will never throw
or actually might (because one callee does throw, or is at a C interface boundary, etc).
One argument against exceptions is that it makes reasoning about control flow harder:
Exceptions more or less force you to reason about the control flow of the whole program at every time.
`noexcept` is supposed to solve that, but, to put that `noexcept` keyword confidently, you still need
to do that analyze. The chances you get it wrong are high.
If you write generic code, you will have to tell the compiler that a symbol is noexcept
if all of it's subexpression is noexcept manually.

And the compiler can not trust you that the function will indeed not throw, so implementers will inject calls to `std::terminate`
here and there, negating somewhat the performance benefits of marking the function `noexcept` in the first place.

Let's rewrite our code using lambda instead

```cpp
auto width = [](const square& s) -> int {
    return s.width;
};
auto area = [](const square& s) -> int {
    return width(s) * width(s);
};
```

Of course, lambdas cannot be forward declared.
So I had to reorganize the code.

And now, despite the lack of `noexcept` keyword,
`static_assert(noexcept(area(square{})));` passes.

**What is happening?**

It turns out that the compiler is pretty good at knowing which functions are `noexcept`.
In the case of lambdas, the definition will always be visible to the compiler before any invocation,
so it can implicitly mark it no except and do the work for us. This allowed as part of C++20.

### What does noexcept even mean?

I'm not saying that `noexcept` would not be necessary in an ideal world, because it has more than one meaning
and people use it differently. Notably, `noexcept` might mean:

* Do not generate exception handling code for this function
* This function does not throw
* This function will _never_ throw

The first statement is a request for the compiler, the second is an assertion for both the compiler and human readers,
while the last one is exclusively for people.

So `noexcept` would remain interesting at API boundary as a contract between people even if the compiler could decide
for itself whether the function was actually non-throwing.

## transaction_safe

The Transactional Memory TS defines the notion of *transaction safe expression* as follow:

> An expression is transaction-unsafe if it contains any of the following as a potentially-evaluated subexpression (3.2[basic.def.odr]):\\

> * an lvalue-to-rvalue conversion (4.1 [conv.lval]) applied to a volatile glvalue\\
> * an expression that modifies an object through a volatile glvalue\\
> * the creation of a temporary object of volatile-qualified type or with a subobject of volatile-qualified type\\
> * a function call (5.2.2 expr.call) whose postfix-expression is an id-expression that names a non-virtual
function that is not transaction-safe\\
> * an implicit call of a non-virtual function that is not transaction-safe\\
> * any other **call of a function, where the function type is not "transaction_safe function"**

(Emphasis mine)

The details are not important, but, basically, a `transaction_safe` safe expression is one that doesn't touch volatile objects.
And only call functions with the same properties.
That's probably upward of 99% of functions - I suspect the very terrible default exists for compatibility reasons.
The important part is that you have to tag all your functions or hope that the property holds true recursively.
(Like `noexcept`, you can lie, by marking a function `transaction_safe` even if a callee is not itself `transaction_safe`, opening the door to UB).
An issue that seems to hold this TS back.

## constexpr

`constexpr` functions are a bit different. The compiler knows what functions are candidate `constexpr`.
Most of the time it will constant evaluate them regardless of whether they are actually marked as such.
The keyword is required to ensure that the compiler will actually do the constant evaluation when it can and, most importantly,
because removing the constexpr-ness of a function may be a source breaking change - (if that function is called during the evaluation of a `constexpr` variable).
By its very nature, `constexpr` implies that `constexpr` functions are defined somewhere is the TU. And everything not defined in the TU cannot be constant-evaluated.
[A proposal for C++20 proposes to make it implicit in some cases](https://wg21.link/p1235)

For now, we are left with the following code, and it is on you to use the appropriate qualifiers.

```cpp
constexpr int width(square s) noexcept transaction_safe;
constexpr int area(square s) noexcept transaction_safe  { return width(s) * width(s); }
constexpr int width(square s) noexcept transaction_safe { return s.width; }
```

As of C++20, `constexpr` functions can throw. The committee is also considering making `new` expressions
`noexcept` by 23 or 26 so we are slowly getting to a place where 95%+ of functions will be both `constexpr` and `noexcept`
eligible and will have to be marked manually.


**Is there a better way ?**

# Back to the C++ compilation model.

A source file and its included headers form a translation unit.
Multiple translations units form a program.

Sounds simple enough right?
It's actually _simpler_ than right.

Headers and sources files are a bit of a lie we tell ourselves.
As far as I can tell, the term "header" only appear in the standard as to name the "standard library headers".
And in practice, headers do not have to be actual files, they identify a thing that can be understood by the compiler
as a sequence of tokens.

In practice, we use the preprocessor - a tech implemented by a drunk bell labs intern on LSD sometime in the late 60s, early 70s -
to stitch together a collection of files that we are never _quite_ sure where in the system they come from.
We call them headers and source files, but really, you can include a `.cpp` file in a `.h` or elect to
use the `.js` extension for headers, `.rs` for sources files and your tools would not care.
You can, of course, create circular header dependencies.

The preprocessor is so dumb you have to tell it explicitly which files it already included with
the crappiest possible pattern called include guard.
This could have been fixed, but you see, it has not because some people are concerned about hardlinking bits of their workspaces together.

In the end, `#include` directives works like `cat` - except `cat` is better as its job.

Oh and of course, because anything can define macros anywhere, any "header" can rewrite all of your code
at compile time in a chaotic manner (here chaotic means deterministic, but well beyond the cognitive capacities of any human being).

In this context, it's easy to understand why the compiler doesn't go look a few ten thousands lines ahead to see whether or not you declared a referenced symbols.
Well, is it a good enough reason?
I don't know...
But, as a consequence (I _think_ this is not really voluntary), Overload and name lookup work as first-good match rather than best match.

```cpp
constexpr int f(double x) { return x * 2; }
constexpr auto a = f(1);
constexpr int f(int x) { return x * 4; }
constexpr auto b = f(1);
```

Pop quiz: What's the value of `a` and `b`?

If you are neither wrong nor aghast, you may be suffering for Stockholm syndrome. There is no cure.
And, because the order of declarations may impact the semantics of a program, and because macros can rewrite everything,
there is no cure for C++ either.

The common wisdom is to put the declarations in headers and the implementations in source files.
That way your very small sources files all including the same hundred thousands lines of header files will compile faster.
At least they will compile less often.
We also established earlier than most code can be constexpr and constexpr declarations must be visible to all translation units.
So, looking at your templated, conceptified constexpr-ified code always using auto, you wonder what you can split up to a source file.
Probably nothing. Unless you stick to C++98 I guess; or make extensive use of type-erasure.
For example, you may use `span`, the best type C++20 has to offer.

And then, of course, the linker will take the various translations units and will
make a program out of it. At this point, the infamous `One Definition Rule` comes into play.
You shall only define each symbol once.
Your hundred of headers expanding to hundred thousands of line of code in various order, with various set of macros defined
in a way specific to that project, on your system, on that day, _shall not_ redefine anything.
Best case scenario you get a linker error. More likely, you get UB. Is your code violating ODR to some extent right now? In all likelihood, it does.
But really, it _shall_ not.
ODR is a direct consequence of your compiler not knowing what names exist in your codebase.

It turns out that Titus Winters talk at length about ODR in great new talk [C++ Past vs. Future](https://www.youtube.com/watch?v=IY8tHh2LSX4).
You should definitively watch this.

**But linkers are pretty great**

They can make static libraries - basically a zip with multiple translations units.
When consuming that library, the linker may conveniently not link otherwise not-referenced static objects.
They didn't get the memo that constructors may have side effects.

They can also make dynamic libraries. The best terrible idea we still believe in.
You probably can get away with making dynamic libraries. It will probably work.
Or not, you will know at runtime.

No, really, linkers _are_ pretty great.

They can optimize the _whole program_ because, unlike compilers, linkers get to see _all of your code_.
So all the code, that you were very careful to split into multiple source files at the expense of a very complicated build system is in the end stitched together by the linker anyway and optimized as a whole that way.

Of course, you are able to run lots of build in parallel, across a distributed build farm, where all of your
gazillion CPU are all parsing `<vector>` at the same time.
The flip side of that is that the compiler itself, expecting you to run multiple jobs at the same time, will not implement any kind of concurrency in its implementation.

What is not used in the call graph starting from the `main()` function or the global constructors is then
thrown away.


# What about modules?

Well, C++ modules help, a tiny bit.

What are C++ modules you might ask? **Standardized precompiled headers is what modules are**.
You get your "headers" in predigested binary form, which makes the compilation faster.
Assuming you don't have to rebuild everything all the time anyway.
I suspect they will really help if you have big third parties implemented in headers.
When tooling figure how to deal with modules.

Note that I believe that modifying a module interface modifies all module interfaces
transitively, even if you don't modify existing declarations.

Importantly, modules are not

* A scoping mechanism, or a way to replace namespaces.

```cpp
//MyFoo.cppm
export module my.foo;
export namespace my::foo {
    constexpr int f() {}
}

//MyBar.cpp
import my.foo;
int main() {
    my::foo::f();
}
```

* A way to permit used-before-declared symbols.

I guess they _could_ have been. Modules being closed, it seems reasonable to consider all the declarations in the same module before doing any parsing of definitions,
but this would make "porting to modules" harder, and "porting to modules" is an important part of the TS.
Unless *_you_* want to write a paper about that?!

* A way to sandbox macros

There is a strong incentive to get modules working on 20yo codebases without actually putting any work into it,
Consequently, the current proposal let you declare and use macros more or less anywhere you want, and possibly export them from modules,
which...I have opinions about. Namely, I think it remains to be seen how modules codebases will actually be built efficiently.

* A way to modernize C++

There have been [some proposals](https://wg21.link/P0997) to disallow or fix some specific constructs in module contexts, I don't expect they will fare well,
once again because people are more concerned about existing codebases than future code.
Python 2 is often used as a cautionary tale in these circumstances.

* Modules

Being glorified compiled headers, C++ modules do not strive to replace the translation units model.
A module is still split as its interface (the compiler can transform the source of that module into a BMI - binary module interface -),
and the definition of the things implemented in the interface (an object file).
In fact, the following code will not link

```cpp
//m1.cppm
export module m1;
export int f() {
    return 0;
}
//main.cpp
import m1;
int main() {
    f();
}
clang++ -fmodules-ts --precompile m1.cppm -o m1.pcm
clang++ -fmodules-ts -fmodule-file=m1.pcm main.cpp
```
because the `m1` module binary *interface* will not consider the definition of `f()`, unless you mark it inline,
or build a .o out of it.
Despite that, the BMI on my system definitively contains the definition of the function, as changing it also changes the
BMI. leading to a rebuild of all dependencies anyway.

So modules are not, a self-sufficient unit like they are in other languages.
Luckily, they do require that the implementation of a given module is done in a single translation unit.

# A set of definitions

People think about their code as a cohesive whole, the colloquial term being a "project".
The more the compiler sees about your code, the more it will be able to optimize it.
An increasing majority of C++ constructs need to be visible to the compiler at all time.
`constexpr` methods, templates (and concepts), lambdas, reflection...

Yet, the compilation model encourages us to make our tools helplessly blind and our lives harder.
The solution to these problems is not trivial.

A core issue is that a program, regardless of the language its written in, is a collection of definitions,
but development tools manipulate files, and there is a certain mismatch there.

For a long time, the C++ community had the deep belief that the separation of definitions and declarations, the source/header model was superior.
But we see an increasing number of header-only libraries, that may be slightly slower to compile but are, at the end of the day,
much easier to use and reason about. For people, for tools, for compilers.
I would not be surprised if future libraries shipped as modules will be "module-interface-only" as well.
I think it does not matter that single-header libraries ship as one file. What matter is that they can be consumed by including a single file.
It expresses "this is the set of declarations that constitute my library."

We should of course not handwave away the problem of long compilation time.
But it is well accepted that most FX/3D artists need a $4000 or more machine to do their job. Studios understand that as the cost of doing business.
And maybe, compiling C++ requires expensive hardware too. And maybe that's okay. Hardware is cheap, people are not. _Especially_ good software engineers.

I don't know if we will ever manage to get rid of object files, static libraries, and dynamic libraries.
I don't know if we will ever stop caring about ABI outside of very specific libraries.

But as the C++ community dreams of better tools, and dependency managers, maybe it would help to define the fundamentals
more accurately: Our programs are a set of _definitions_, some of which are provided and maintained out-of-tree by other people.
I think the more closely our tools adhere to that model, the better off we will fare in the long run.

So maybe we need to ask fundamental questions about the compilation model and examine
some beliefs we hold (For example "Compilers and build system need to be kept separated". Do they? To what extent?).

There are definitively immense technical roadblocks, social and legal ones (LGPL, you should be ashamed of yourself).
It seems impossible, but the reward would be, ô so great. In the meantime, fully aware I don't have any answer, I'll shout on the Internets.