---
title: "Undefining the C++ Pre-processor"
date: 2018-01-10
---

There are only two kinds of languages: the ones people complain about and the ones nobody uses — Bjarne Stroustrup

I like that quote. it explains both JavaScript and Haskell. And by that measure the preprocessor is a great language in that people use it, a lot. It’s never considered separately from C and C++, but if it was, it would be the number one language on [TIOBE](https://www.tiobe.com/tiobe-index/). The preprocessor is both extremely useful, and pervasive. Truth is, it would be *really* hard to write any kind of serious and portable C++ application without the preprocessor being involved at some point.
> — The preprocessor sucks
> — I know, right? It’s the worst. Hey, can you merge my commit ? I added a bunch of useful macros.

I think a lot of people are familiar with that kind of conversation, and if we are not careful, we may still have them 20 years from now. Because existing is, unfortunately, the only redeeming quality of the preprocessor. Alas, my issues are neither theoretical, philosophical nor idealistic.

I don’t care at all that the preprocessor let anyone replace identifiers, keywords (some say, that’s illegal, in practice…) without any kind of check. Neither do I care that the preprocessor manages to be [Turing complete](http://www.ioccc.org/2001/herrmann1.hint) while not being able to handle commas properly. I don’t even care about includes and includes guards, and I haven’t a single issue with#pragma. Sometimes you have to be pragmatic.

However.

Let me offer you a scenario, you might find it contrived but please bear with me. So, Imagine that you are refactoring some cross platform application and you decide to do something unusual like, say, renaming a function.

That’s not possible. Never have been, probably never will be.

```cpp
#ifdef WINDOWS
  foo(43);
#else
  foo(42);
#endif
```


Fundamentally, neither the compiler nor your tools ( a tool being by necessity a full fledged compiler front end ) have a full view of your code. The disabled parts are not compiled, parsed, lexed or otherwise analyzed.

First, the disabled paths have no obligation to be valid C++. This is valid:

```cpp
#if 0
#!/bin/bash
    g++ "$0" && ./a.out && rm ./a.out
    exit $?;
#else
#include <iostream>
int main() {
    std::cout << "Hello ?\n";
}
#endif
```

So if the compiler were to take into account the disabled paths of the preprocessor it may not be able to for a valid AST. Worse, preprocessing, as the name suggests, happen as a separate state and a preprocessing directive may be inserted between any two C++ token including in the middle of any expression or statement.

```cpp
#if 0
  void
#else
  bool
#endif

#if 0
  &
#endif
#if 0
  bar(int
#else
  baz(long,
#endif
#if 0
  , std::vector<
#   if 0
      double
#   else
      int
#   endif
    >)
#else
  double)
#endif
;
```

The other equally concerning issue is that the compiler can not possibly know what combination of #ifdefand #defines statements are supposed to form a valid program.

As an example, Qt offers a set of defines that can be set to enable or disable certain features of Qt at compile time. Say you don’t need a calendar widget, you can define #QT_NO_CALENDAR_WIDGET and that makes for a smaller binary. It doesn’t work. I suspect it *never* worked. See, At some point Qt had about 100 such compile time configuration options. Given that the number of build configurations possible explodes exponentially with the number of variables. when you may have 2¹⁰⁰ variation of your program, automation proves difficult , even at big-web-deep-cloud-hexa scale.
> Untested code is broken code.

You probably know that famous adage. So what about not even compiled code ?

I should point out that putting some platform specific method in platform specific files leads to the exact same issue. Basically the code that the compiler see should be a single self contained source of truth, but instead the code is fragmented and the vision that you have of it is, as best, incomplete.

## The preprocessor is considered harmful, what can we do about it ?

By the way, it’s not just the preprocessor that is flawed. So are all modern processors apparently. Maybe anything doing some sort of processing should be avoided ?

Anyway, let’s see what we can do about preprocessor directives, today.

### 1. Strongly Prefer constants over #define

This one is simple enough, yet I still see a lot of constants defined using macros. Always use static const or constexpr rather than a define. If your build process involves setting a set of variable such as a version number or a git hash, consider generating a source file rather than using defines as build parameters.

### 2. A function is always better than a macro

```
#ifndef max
#define max(a,b) ((a)>(b)?(a):(b))
#endif
#ifndef min
#define min(a,b) ((a)<(b)?(a):(b))
#endif
```

The above snippet is from the **Win32 API**. Even for “simple” and short one liner you should always prefer a function.

If you need lazy evaluation of the function arguments, use a lambda. Here is a solution that ironically, uses macro, but it’s a start !
[Lazy evaluation of function arguments in C++] (http://foonathan.net/blog/2017/06/27/lazy-evaluation.html)

### 3. Abstract away the portability concerns.

Properly isolating the platform-specific nastiness in separate files, separate libraries and methods should reduce the occurrence of #ifdef blocks in your code. And while it does not solve the issues I mentioned above you are less likely to want to rename or otherwise transform a platform-specific symbol while not working on that platform.

### **4. Limit the number of variations your software can have.**
> Should that dependency really be optional?

If you have optional dependencies that enable some feature of your software considering using a plugins system or separate your projects in several, unconditionally build components and applications rather than using #ifdef to disable some code paths when the dependency is missing. Make sure to test your build with and without that dependency. To avoid the hassle, consider never making your dependency optional
> Should this code really only be executed in release mode ?

Avoid having many different Debug/Release code paths. Remember, not compiled code is broken code.
> Should that feature really be deactivatable ?

Even more so than dependencies, features should never be optional at compile time. Provide runtime flags or [a plugin system](http://www.boost.org/doc/libs/1_66_0/doc/html/boost_dll.html).

### 5. Prefer pragma once over include

Nowadays, the exotic C++ compilers that don’t support #pragma once are few and far between. Using #pragma once is less error-prone, easier and faster. Kiss the include guards goodbye.

### 6. Prefer more code over more macro

While this one is to be adapted to each situation, in most cases it’s not worth it to replace a few c++ tokens with a macro. Play within the rule of the language, don’t try to be overly clever and tolerate a bit of repetition, it will probably be as readable, more maintainable, and your IDE will thank you.

### 7. Sanitize your macros

Macros should be undefined with #undef as soon as possible. never let an undocumented macro in an header file.

Macros are not scoped, use long uppercase names prefixed with the name of your project.

If you are using a third party framework such as Qt that have both short and long macro names ( signal and QT_SIGNAL ), make sure to disable the former, especially if they may leak as part of your API. Don’t offer such short names yourself. A macro name should stand from the rest of the code and not conflict with boost::signal or std::min

### 8. Avoid putting an ifdef block in the middle of a C++ statement.
```cpp
foo( 42,
#if 0
    "42",
#endif
    42.0
);
```

The above code has a a few issues. It’s hard to read, hard to maintain and will cause issues to tools such as clang-format. And, it also happen to be broken.

Instead, write two distinct statements:

```cpp
#if 0
    foo(42, "42", 42.0);
#else
    foo(42, 42.0);
#endif
```

You may find some cases where that’s hard to do, but that’s probably a sign that you need to split your code into more functions or better abstract the thing you are conditionally compiling.

### 9. Prefer static_assert over #error

Simply use static_assert(false)to fail a build.

## The preprocessor of the future past

While the previous advice apply to any C++ version there is an increasing number of ways to help you reduce your daily intake of macros if you have access to a fresh enough compiler.

### 1. Prefer modules over includes

While modules should improve compile times they do also offer a barrier from which macros cannot leak. In the beginning of 2018 there are no production ready compiler with that feature but GCC, MSVC and clang have implemented it or are in the process to.

While there is a collective lack of experience, it’s reasonable to hope that modules will make tooling easier and better enable features such as automatically including the module corresponding to a missing symbol, cleaning unneeded modules…

### 2. Use if constexpr over #ifdef whenever possible

When the disabled code-path is well-formed (does not refers to unknown symbols), if constexpris a better alternative to #ifdef since the disabled code path will still be part of the AST and checked by the compiler and your tools, including your static analyzer and refactoring programs.

### 3. Even in a postmodern world you may need to resort to an #ifdef, so consider using a postmodern one.

While they don’t help solving the issue at hand, at all, [a set of macros](http://en.cppreference.com/w/cpp/experimental/feature_test) is being standardized to detect the set of standard facilities offered by your compiler. Use them if you need to. My advice is to stick to the features offered by every and all compilers your target. Chose a baseline an stick with it. Consider that it might be easier to back-port a modern compiler to your target system than to write an application in C++98.

### 4. Use std::source_location rather than __LINE__ and __FILE__

Everybody like to write their own logger. And now you can do that with less or no macro using [`std::source_location`](http://en.cppreference.com/w/cpp/experimental/source_location).

## The long road towards macro-free applications

A few facilities offer better alternatives to some macro usages, but realistically, you will still have to resort to the preprocessor, sooner than later. But fortunately, there is still a lot we can do.

### 1. Replace -D with compiler-defined variables

One of the most frequent use case for defineis to query the build environment. Debug/Release, target architecture, operating system, optimizations…

We can imagine having a set of constants exposed through a std::compiler to expose some of these build environment variables.
```cpp
if constexpr(std::compiler.is_debug_build()) {  }
```

In the same vein, we can imagine having some kind of extern compiler constexpr variables declared in the source code but defined or overwritten by the compiler. That would only have a real benefit over constexpr x = SOME_DEFINE; if there is a way to constrain the values that these variables can hold.

Maybe something like that
```cpp
enum class OS {
    Linux,
    Windows,
    MacOsX
};

[[compilation_variable(OS::Linux, OS::Windows, OS::MacOsX)]]  extern constexpr int os;
```

My hope is that giving more information to the compiler about what the various configuration variables are and maybe even what combination of variables are valid would lead to a better modeling (and therefore tooling and static analysis ) of the source code.

### 2. More attributes

C++ attributes are great and we should have more or them. [[visibility]] would be a great place to start. it could take a constexpr variable as argument to switch from import to export.

### 3. Taking a page from Rust’s book

The Rust community never misses an occasion to promote fiercely the merits of the Rust language. And indeed, Rust does a lot of things really well. And compile time configuration is one of them.

```rust
// The function is only included in the build when compiling for macOS
#[cfg(target_os = "macos")]
fn macos_only() {
  // ...
}

// This function is only included when either foo or bar is defined
#[cfg(any(foo, bar))]
fn needs_foo_or_bar() {
  // ...
}
```

Using an attribute system to conditionally include a symbol in the compilation unit is a very interesting idea indeed.

First, it’s really readable and self documenting. Second, even if a symbol is not to be included in the build, we can still attempt to parse it, and more importantly, the sole declaration gives the compiler sufficient information about the entity to enable powerful tooling, static analysis and refactoring.

Consider the following code:
```cpp
[[static_if(std::compiler.arch() == "arm")]]
void f() {}


void foo() {
    if constexpr(std::compiler.arch() == "arm") {
        f();
    }
}
```

It has an amazing property : It’s well formed. Because the compiler knows that f is a valid entity and that it is a function name, it can unambiguously parse the body of the discarded if constexpr statement.

You can applies the same syntax to any kind of C++ declaration and the compiler would be able to make sense of it.
```cpp
[[static_if(std::compiler.arch() == "arm")]]
int x = /*...*/
```
Here the compiler could only parse the left hand side since the rest is not needed for static analysis or tooling.
```cpp
[[static_if(std::compiler.is_debugbuild())]]
class X {
};
```

For static analysis purposes we only need to index the class name and its public members.

Of course, referencing a discarded declaration from an active code path would be ill formed, but the compiler could check that it *never* happens for any valid configuration. Sure, it wouldn’t be computationally free but you would have a strong guarantee that *all* of your code is well formed. Breaking the windows build because you wrote your code on a Linux machine would become much harder.

It’s however not easy as it sounds. What if the body of discarded entities contains syntax the current compiler doesn’t know about ? Maybe a vendor extension or some newer C++ feature ? I think it’s reasonable that parsing happens on a best-effort basis and when a parsing failure happens the compiler can skip the current statement and warns about the parts of the source it doesn’t understand. “I haven’t been able to rename Foo between lines 110 and 130” is miles better that “I have rename some instances of Foo. Maybe not all, good luck skimming through the whole project by hand, really don’t bother with a compiler, just use grep”.

### 4. constexpr all the things.

Maybe we need a constexpr `std::chrono::system_clock::now()` to replace `__TIME__`

We may also want a [compile time Random Number Generator](https://www.youtube.com/watch?v=rpn_5Mrrxf8). Why not ? Who cares about reproducible builds anyway ?

### 5. Generate code and symbols with reflection

The metaclasses proposal is the best thing since sliced bread, modules and concepts. In particular [P0712](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0712r0.pdf) is an amazing paper on many regards.

One of the many constructs introduced is the declname keyword that creates an identifier from an arbitrary sequence of strings and digits

`int declname("foo", 42) = 0;` creates a variable `foo42` . Given that string concatenation to form new identifiers is one of the most frequent use case for macros, this is very interesting indeed. Hopefully the compiler would have enough information on the symbols created ( or refereed to ) this way to still index them properly.

The infamous [X macro](https://en.wikipedia.org/wiki/X_Macro) should also become a thing of the past in the coming years.

### 6. To get rid of macros, we need a new kind of macros

Since macro are just text replacement, their arguments are lazily evaluated. And while we can use lambda to emulate that behavior, it’s rather cumbersome. So, could we benefit from lazy evaluation in functions ?

This is a topic I thought about last year
[Research on code injection & reflection in c++](https://github.com/cor3ntin/CppInjectionReflection)

My idea is to use the facilities offered by code injection to create a new kind of “macros” which I call “syntactic macros” for lack of a better name. Fundamentally, if you give a name to a code fragment ( a piece of code that you can inject at a given point of your program), and allow it to take a number of parameters, you’ve got yourself a macro. But a macro which is checked at the syntax level (rather than the token source the preprocessor offers).

How would it work ?

```cpp
constexpr {
    bool debug = /*...*/;
    log->(std::meta::expression<const char*> c,  std::meta::expression<>... args) {
            if(debug) {
                -> {
                   printf(->c, ->(args)...);
               };
         }
    }
}

void foo() {
     //expand to printf("Hello World")  only and only if debug is true
    log->("Hello %", "World");
}
```

Ok, What’s happening here.

We first create a constexpr block with `constexpr { }`. This is part of The meta class proposal. A constexpr block is a compound statement in which all the variables are constexpr and free of side effects. The only purpose of that block is to create injection fragments and modify the properties of the entity in which the block is declared, at compile time. ( **Metaclasses** are syntactic sugar on top of **constexpr** blocks and I’d argue that we don’t actually need metaclasses.)

Within the constexpr block we define a macro log. Notice that macro are not functions. They expand to code, they don’t return anything nor do they exist on the stack. log is an identifier that can be qualified and can not be the name of any other entity in the same scope. Syntactic macros obey the same lookup rules as all other identifier.

They use the `->` injection operator. `->` can be used to describe all code injection related operations without conflicting with its current uses. In your case since log is a syntactic macro which is a form of code injection, we define the macro with `log->(){....}`.

The body of the syntactic macro is itself a constexpr block which may contain any C++ expression that can be evaluated in a constexpr context.

It may contain 0, one or more **injection statements **denoted by `-> {}` . An injection statement creates a code fragment and immediately inject it at the point of invocation, which is, in the case of the syntactic macro, the location where the macro is expanded from.

A macro can either inject an expression or 0 or more statements. A macro that inject an expression can only be expanded where an expression is expected and reciprocally.

Though while it has no type, it has a nature which is determined by the compiler.

You can pass any arguments to a syntactic macro that you could pass to a function. Arguments are evaluated before expansion, and are strongly typed.

However, you can also pass reflections on an expression. That suppose being able to take the reflection of arbitrary expressions. A reflection on an expression ehas a type corresponding to decltype(e).

In term of implementation, in the above example above std::meta::expression<char*> is a concept matching any reflection on an expression which type is char*.

The last piece of magic when evaluating a macro is that expressions are implicitly converted to their reflection before expansion.

At a basic level, we are moving AST nodes around, which is consistent with the current approaches on reflection and code injections.

Lastly, when we inject `print(->c, ->(args)...)` notice the `->` tokens. That transform the reflection back to the original expression which can then be evaluated.

From the call site, `log->("Hello %", "World");` looks like a regular void function call except that the `->` indicate the presence of a macro expansion.

Lastly, the ability to pass as argument an identifier before evaluation may alleviate the need for new keywords:

`std::reflexpr->(x)` could expand to `__std_reflexpr_intrasics(x)` before `x` is evaluated.

### Do S-Macro replace preprocessor macros completely ?

They don’t, but they don’t intend to. Notably, because they must be valid c++ and are checked at multiple point ( at definition time, before, during and after expansion) they actively prohibit token soup. They are valid C++, inject valid C++ and use valid C++ as parameters.

That means that they can’t inject partial statements, manipulate partial statements or take arbitrary statement as parameters.

They do solve the issue of lazy evaluation and conditional execution. For example you can not implement foreach with them since `for(;;)` is not a complete statement ( `for(;;);` and `for(;;){}` are but they are not very useful).

There are a lot of questions regarding name lookup. Should a macro “see” the context it’s expanded in ? Should and argument be aware of the inner of the macro ? it’s declaration context.

I think limitations are a good thing. If you really need to invent new constructs, maybe the language is lacking, in which case, write a proposal. Or maybe you need a code generator. Or just more abstractions, or more actual code.

### Is this real life ?

It’s very much fantasy and absolutely **not** part of any current proposal, but I do think it would be a logical evolution of the code injection feature.

It resembles a bit to [rust macros](https://doc.rust-lang.org/1.10.0/book/macros.html) — except it does not allow for arbitrary statements as arguments — while (I hope) feeling like part of C++, rather than being another language with a separate grammar.

The Preprocessor certainly looks like a fatality. But there is a lot of things you can do to depend less on it. And there is a lot that the C++ community can do to make macros increasingly less useful by offering better alternatives.

It may take decades, but it will be would be worth it. Not because macro are fundamentally bad, but because tooling is and will be more and more what languages are judged on, live and die bad.

And because we sorely need better tooling we need to do whatever we can to diminish our fatalistic reliance on the preprocessor.

`#undef`
