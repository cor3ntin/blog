---
title: "Modules are not a tooling opportunity"
date: 2018-10-31T23:16:15+01:00
---

C++ Modules are going through the standardization process and current plans would have them merged in the C++ Standard in time for C++20.
They are a great language feature, offering a number of benefits over headers

* They feel more modern
* They are _much_ faster to parse
* They provide protections against macros
* They provide some protections against ODR violations.

I really can't wait to be able to replace headers with them in my code bases.
Still, I have a few concerns with them and think they could go further [In replacing the classic translation unit model](posts/translation_units/).
I'm afraid that the "legacy" features that add a ton of complexity to the design will never be legacy,
and will be a long-term source of issues for the benefits of short-term gains. I may be wrong and I certainly hope I am.

But, what concerns me the most is the question of how tooling and modules will integrate with each other, an issue that I feel has been somewhat handwaved away.
The paper [C++ Modules Are a Tooling Opportunity](wg21.link/p0822) call for better tools. It's hard not to agree with the author.
C++ tooling, is, for the most part, stuck in the past.

It is however very optimistic to think modules will magically lead to better tooling. Namely, modules can hardly lead to better build systems.
Build systems don't have much value for the final product leading companies to either

* Organically grow a set of scripts over decades, they barely work but no one wants to pay a few engineers for months to upgrade them to a better solution
* Use an existing solution to benefit from a wider ecosystem.

This pattern has led to the adoption to `CMake` (a bunch of scripts that barely work but with the benefit of a wide ecosystem) by a large number of products.
There also exist a number of more modern build systems that fail to reach a critical mass before their maintainers loose faith, and are [simply abandonned](http://blog.qt.io/blog/2018/10/29/deprecation-of-qbs/) or used by 3 people in their basement.

Growing a new build system takes years and represent a significant investment, it's not something that can be wished to existence.
No matter how much I would like that promised magic tool.


More importantly, the challenges tools (Build systems, IDEs, refactoring, indexing, etc) face in order to handle module smoothly are independent of the age or quality of the tool.
The problem is simple.
Imagine you have 100s/1000s of modules. Maybe more.
You don't have to be a big company to have that many modules. Maybe you wanted to add a small feature to LLVM or chrome.
Or maybe you use `vcpkg` to handle a large number of dependencies. Why reinvent the wheel when all this beautiful code exist?

You create a bunch of files for a new project

```cpp
//foo.cppm
export module cor3ntin.foo;
export import cor3ntin.foo.bar;
import google.chrome.net.http;

export namespace cor3ntin {
    inline void this_is_not_important() {}
}

//bar.cppm
export module cor3ntin.foo.bar;

//main.cpp
import cor3ntin.foo;
int main() {
    cor3ntin::this_is_not_important();
}
```
This actually look rather elegant and modern, even if these things are somewhat subjective.
It is important to note a couple of things

* My modules are called `cor3ntin.foo`: The `.` has no intrinsic meaning: modules are **not** hierarchical, but for the sake of one day have a nice ecosystem it is important to behave like if they were.
  By having an organization name as part of your modules name you ensure uniqueness across your project and its dependencies. No one forces you to do that, but, please do it?
* The first thing I do is open a namespace called like part of the module name. Modules are not a namespacing mechanism. It kinda makes sense in the C++ world because of legacy and some differences between
namespaces and modules, but it surprises a lot of people (I was surprised too at first) because it is contrary to what is done is a lot of other languages


You also have a CMakeFile.

```cmake
add_executable(foo
               main.cpp
               foo.cppm
               bar.cppm
)
target_link_library(foo PUBLIC google-chrome::net)
```

And you ask Cmake to run the build. Or rather to generate a script for an even more ill-equipped tool that will run the build.
I imagine cmake will see that `main.cpp` depends on nothing, so that's the first thing it will put in the dependency graph.

```
> compilator3000 main.cpp -o main.o
Error: no module named cor3ntin.foo
```

Because of course, at this point the module binary interface it is looking for has not been precompiled yet. How do we fix that?

# Manually expressing the dependencies graph

Well, an obvious solution is to build a dependency graph for all your modules manually.

```cmake
add_cpp_module(bar-module bar.cppm)
add_cpp_module(foo-module foo.cppm DEPENDS bar-module google-chrome::net-http-module)
add_executable(foo
               main.cpp
               foo-module
               bar-module
)
target_link_library(foo PUBLIC google-chrome::net)
```

This is not currently valid `CMake` syntax, but hopefully, you can understand what it would do: explicitly create a target (graph node) for each module.
And while cmake has no support for modules, this kind of manual way of expressing the dependency graph is how modules seem to have been used by companies
who tested the module TS.

With that cmake can do things in the correct order:

* Build `google-chrome::net-http-module` so we can import the `google.chrome.net.http` BMI
* Build `bar-module` so we can import the `cor3ntin.foo.bar` BMI
* Build `foo-module` and importing the now existing BMI `cor3ntin.foo.bar` and `google.chrome.net.http`
* build main.cpp
* Build the executable

So, it would work. And maybe there is an expectation that modules will be used that way.

When I was about 2 weeks old, my mom told me to avoid duplication. She explained it was good engineering practice.
It made perfect sense and I strive to avoid code duplication ever since.
And other people seem to think that too because they invented generic programming, templates and [even functions](https://www.youtube.com/watch?v=2jeJixIQlx4)
just to get closer of that goal of expressing themselves with no duplication.

As an industry, we know that code duplication leads to harder to maintain code and we like our code to be maintainable because we are nice people.
We especially like to be nice to our future selves.

Modules are no different. Putting our code in well-delimited unit of works, that are reusable and shareable, is a way to avoid code duplication.

Why I'm telling you all of that?
Well, let's look at our project.

We have a file `foo.cppm`. It declares a `cor3ntin.foo` module. Which is built by the `foo-module` target?
This is saying the same thing 3 times. With different names. And, as the saying goes, the 3 hardest problems in computer science are naming and consistency.

More critically, we have duplicated the dependencies of our modules.
`add_cpp_module(... DEPENDS bar-module)` in the build script encodes the exact same information as `import cor3ntin.foo.bar;` in the source file.
Meaning each time we want to add or remove a module from a file we to edit the build script.

(Notice also that I have not specified build flags for individual modules, but that would need to be added too, presumably leading to more duplication or complexity)

If you have hundreds of modules or have to rewrite a dependency's build script, this scheme is really not maintainable. And it makes `modules` somewhat not appealing.
The last thing I want or need is more build scripts.

# Automatic dependency graph building

Instead, what we really want is to go back to the simplicity of our first `CMakeFiles`
```cmake
add_executable(foo
               main.cpp
               foo.cppm
               bar.cppm
)
target_link_library(foo PUBLIC google-chrome::net)
```

And, will make `cmake` smart. It's a tall order but bear with me.
Cmake will open all the files and lex them to extract the list of dependencies of every module.

Main: not a module declaration, but depends on `cor3ntin.foo`
foo.cppm : this is a module called `cor3ntin.foo`, it depends on `cor3ntin.foo.bar` and `google.chrome.net.http`. Add it to the dependencies of `main.cpp`
bar.cppm : this is a module called `cor3ntin.foo.bar`. Add it to the dependencies of `foo.cppm`

CMake also has to parse the entirety of Chrome's code base to find a file declaring `google.chrome.net.http`.

To do that, it has to open each file, and preprocess a "preamble" which may contain macros, and include directives. Conditionally import code, etc.
So, it takes a while. Also, the parsing has to be accurate, so you need to defer to a full-fledged compiler to get the actual dependencies, which is _slow_.
Maybe vendors will be able to provide a library to resolve dependency without having to open a process. One can certainly hope!
Or maybe P1299, which argue in favor of `import` declarations _anywhere_ in the global scope will be adopted in which case,
cmake will have to preprocess and lex all of your c++ all of the time.

After a while, CMake has in memory the dependency graph of all modules of the chrome codebase and ours, even if we only care about the dependencies of `google.chrome.net.http`.
This has to be cached, so the build system needs to be stateful, which I don't think is a source of issues, but it's worth pointing out.

At this point, you have a dependency graph and you can start doing your builds and dispatch things to build nodes if you are fancy at scale. Which, to be clear, a lot of companies
have to be. I don't think google's code base would build on my laptop in a reasonable time frame.

Let say you modify `foo.cppm`. Your build system needs to see that and rebuild everything it needs.
As an apart, let me tell you about the two kinds of build systems there are:

* Build systems that, upon a change in the codebase will always run the minimum and sufficient set of tasks to update the artifacts as to apply these changes.
* Build systems that are garbage. Expect more of your tools!

But many things may have happened:

* You renamed the module (changed `export module cor3ntin.foo` to `export module cor3ntin.gadget` )
* You added an import

And you might have done that to _any_ modified file

So, your build tool has to lex all your modified files again. And rebuild the dependency graph again. In the cmake world, that means running cmake again.
The generators are simply not able to handle that

Modifying your source code modify the dependency graph in all sort of ways. Which is is very new. I think it's also very cool because when it works it lets you focus on code
rather than on translation units and build systems.

But, on the flip side, you have to operate a full scan of modified files every single time you compile. On your computer, on the build farm, everywhere.
Which maybe takes 5 seconds, maybe takes a few minutes.
And if your code is fully modularized, which I hope it will be in a few years, the build system will likely have little to do until that scan is complete.

Ok, enough talks about build systems, let's talk about IDEs.

You decide to modify `main.cpp`, so you open your project in an IDE. Maybe Qt Creator, VS, VSCode, emacs... whatever tickles your fancy.
That ide would like to offer completion because it's nice. And also, you know, that's what IDEs are for.
So, your IDE goes on a quest to a list of all the symbols in all the imported modules.
Modules are not portable, so the IDE will try to read the source file of the modules instead.
It sees that you imported a module `cor3ntin.foo` so it starts to frantically lex all the files of your project and its dependencies until it finds one
that declares the appropriate module. It has to do that for every and all import declaration.
Your MacBook is now so hot you discover a new state of matter. And, hopefully, after a few minutes, you have a usable symbol index

Or maybe the IDE defers to an external symbol server such as `clangd`. Which require a compilation database. Which has to be rebuilt every time a source change.

In fact, any kind of tool that needs to index symbols or run static analysis or anything else will either need to have access to the precompiled BMIs of all your import or be
able to map a module name to a file name.

# Possible solutions to the tooling issuies

## Module map

The no-longer pursued clang proposal for modules has a "module map" file that map a module name to a file name.
This is morally equivalent - albeit more portable - than declaring all your modules explicitly in a build script.
There is still a lot of duplication and the risks of things not kept in sync

## Module mapping protocol

[P1184](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p1184r0.pdf) proposes a protocol such that the compiler can query the build system and ask the location of a BMI matching a given name.
This is somewhat different because presumably, it would allow you to run all your compilation at one, which is desirable in a parallel system
and then each compilation will presumably be idle until the build system notifies them that a BMI is available.
Very cautious not to turn our compilers into build systems, we are considering turning them into servers.

What could possibly go wrong 👻 ?

Such a system specifically does not work with a meta build system such as cmake.
Personally, I hate meta build systems, so I would not mind, but, it's worth keeping that in mind.

## Put the name of the module in the name of the file that declares it.

This is my favorite solution. I think it was discussed and rejected.

The idea is simple.
Instead of having a file `foo.cppm`, requires that the file encodes the module name `cor3ntin.foo.cppm`. And make `.cppm` a mandated extension for modules.
Such that:

* The build system can assume which files are modules and which are not.
* Upon encountering `import cor3ntin.foo`, we know immediately what files to scan next.

This is particularly desirable for tools other than build systems, but it also helps build systems construct the dependency graph in an orderly fashion, which means
that individual tasks can be scheduled sooner and more predictably.
When a module interface file is edited, it may still modify the graph, but only add or remove vertices to the one node corresponding to that file.

From a performance standpoint, scanning directories is much faster than lexing c++. Although performance is still a concern on windows, where scanning files is routinely 10x slower than on most other mainstream OS.

It solves the duplication issue, although most languages elect to have the information in both the source file and the file name, most likely for robustness.

### Some drawback of this proposal

* I would expect some bikeshedding over whether it should be encoded in the path or the filename, which doesn't really matter given modules have no semantical notion of hierarchy.
* It might be considered out of scope of wg21 because naming files fall outside of the scope of a language, right?
  Well, I guess that's true, except if you ignore the languages that do have semantically meaning full filenames:
    - Java
    - Python
    - Haskell
    - Erlang
    - D
  A certainly a few others.

### The Woodstock approach to standardization

A lot of people seem to see the benefit of imposing some structure in the name or path of files declaring module interface.
But they think it should be left to the vendors.
The hope is that vendors of all the myriad of build systems, IDE and tools will come together and agree on a similar solution for similar reasons, with the power of... flowers, I guess.
Which is great, but isn't C++ a standard because we know from experience this has absolutely no chance of working?
And remember. The dream of a universal dependency manager can only come to life if we speak a common language.

The standard would not even have to mention files. I guess something along the line of 'a module name `X` identifies a unique module declared by a resource `X.cppm`', would work.

# More issues with modules

This is I think the major issue with modules but it is not the only one.
For example, I don't think anyone knows how legacy headers are possibly toolable at the build system level.
The module format is also not restricted at all.
Which means that the build system behavior may depend on specifics compilers. For example, Microsoft BMI are more optimized than Clang's, so clang might trigger more rebuilds.


# Where to go from there?
Modules will be discussed at San Diego. And they are great. They could be much better.

But until we have a better picture of their integration with build systems and tools and the certitude that they deliver
the build time gains the promised on both small and large project... I will remain cautiously pessimistic

# Further Reading

* [Remember the FORTRAN](https://wg21.link/P1300)
* [Implicit Module Partition Lookup](https://wg21.link/P1302)
* [Merged Modules and Tooling](https://wg21.link/P1156)
* [Response to P1156](https://wg21.link/P1180)
* [Module Preamble is Unnecessary](https://wg21.link/P1299)
* [Impact of the Modules TS on the C++ tools ecosystem](https://wg21.link/p0804)
* [C++ Modules Are a Tooling Opportunity](https://wg21.link/P0822R0)
* [Building Module - Youtube](https://www.youtube.com/watch?v=E8EbDcLQAoc)
* [Progress With C++ Modules - Youtube](https://www.youtube.com/watch?v=5CadIjPRZpM)