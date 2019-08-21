---
title: "Towards Better C++ Modules - Part 1: Modules mapping"
date: 2018-11-26T20:26:40+01:00
---

In this blog post, we will talk about modules mapping.
Modules mapping is a mechanism by which is a module name is associated with the source file that defines that module's interface.

A module is closed and self-contained.
Which mean that for every module name there must exist one and only source file defining that module.

Module mapping is not useful to the compiler per-say.
By the time the compiler encounters an `import` declaration, the module _binary_ interface must exist and be known
to the compiler.

However, module mapping is very important to _tooling_. It is notably something build systems will have to perform
constantly since module mapping is necessary to build the dependency graph.

Remember that modules make the dependency graph more dynamic and notably the dependency graph needs to be refreshed
every time a file is modified.


# Module, where you at?

Before we can find one module, we need to find all modules.
Every time a header exists in an include world, a module may exist in an import universe.

 * Your code will both consume and produce modules, just like it uses and produces headers.
 * The STL will most likely be a snowflake module - that will just be there
 * System libraries may use - Why would they not? So all the development packages in Debian might have module interface.
 * Non-System third-party libraries - Maybe these are in a git submodule, Conan, Vcpkg, a folder somewhere on a shared drive mounted from that computer in Dennis's office downstairs.
 * Maybe even the module you are looking for isn't even in your computer at all. Because if you write `import sg15.io2d` your awesome build system will fetch the corresponding module on the internet for you. That's a story for another time.

In short, while there are some expectations that modules will be easier to collect than headers since modules don't suffer the issues related to paths management,
a builds tool will have to look in a number of places to collect a *list of files that may declare a module*.

Armed with a list of places where you might find files that may declare modules, we must collect individual module-declaring files.
A simple way to do that is to look at each file's extension.
Might a `.h` declare a module ? A `.hpp` ? A `.hppm` ? A `.cpp` ? A `.cppm` ? A `.cxx` ? `.mpp` ? `.mxx` ?
The thing is, The standard doesn't concern itself with file extensions, so a build system,
one that will scan files for you will have to poke at anything that might possibly declare a module. And yeah, that probably means all existing `.h` and `.hpp`
out of habit, because nobody will tell them to, people will write libraries that use this scheme.


# Poking at modules

To get the name of the modules declared in a given file, you must open it and preprocess and lex it until you get an `export module name;` declaration.
This can be hundreds of lines into the file and the file might also declare a module global fragment which the build system doesn't care about -
but which need possible for modules to include non-modular code.
I will come back to the preprocessor in a later article.
For now, it's enough to say that extracting the name of a module from a file is non-trivial and require a full-fledged compiler.

And, if a translation unit, for example, depends on a module `foo`, you may have to open hundreds of files, until you find one that declares `foo`.
On some system, opening files and launching process can be costly and so mapping a module to a file may take a while.

You might argue that the same problem exists for dependency extraction.
And that's true, files need to be open, preprocessed and lexed in order to extract build dependencies.

But there are other use cases to consider:
For example, An IDE will need to be able to do a quick-mapping in order to provide completion for a single-translation unit.
Tools providing completion, metrics on dependencies (including package manager), etc will have to provide that mapping.

To be clear, module<->file mapping isn't the biggest toolability concern of modules, but it is one.

# Easier Mapping

A few solutions have been proposed to make it easier for tooling to map a name to a file.

## Manually describe the mapping in the build system

The idea is to let developers describe modules in the build system directly.
For example, if you use cmake, you could write:
```cmake
    add_module(foo, foo.cppm)
```

But this is not about cmake, for example, `build2` supports exactly that
```
    mxx{foo}@./: cxx.module_name = foo
```

This is a bit cumbersome, as one might have hundreds of modules.
It also duplicates information (Modules names are encoded in source files and in the build systems).
It forces you to know what modules each of your dependencies use and  in general, makes it very hard to migrate from one build system to another, or for example
use a library originally wrote with Meson in a Bazel build.

## Standard-ish Module Mapping file

The idea is a bit similar to describing the mapping in the build system, but instead of putting the mapping in a `CMakeLists.txt` or `Makefile`,
you would put it in another file that whose syntax would be specified in a Standing Document
(in the hope of making it an industry standard even though it would not be standard).

Conceptually this file would be very simple:

```
foo: foo.cppm
bar: bar.mpp
```

This solves the issue of portability across build system. But the other issue remains: The module name is still duplicated.
This also poses interesting challenges: For example, how to handle modules generated during the build?
But more importantly, where are these files located within the source tree of third parties? How do they work on package-based systems such as Debian?

## Standard layouts.

[A paper](https://wg21.link/P1302) proposes that module mapping can be encoded as part of the file _path_ where `core.io` would map to `core/io.cxx`.
There are a few issues with that design
 * While filesystems are understood to be hierarchical, modules are not. Remember that despite `.` being a valid character within a module identifier, it has no semantic meaning.
`core` is not necessarily a superset of `core.io`
 * It's unclear to me how that system would work with external and system libraries
 * It can not be enforced
 * People would argue about which layout is the best and we would get nowhere. It actually what happened at San Diego. People do not want to adapt a layout, even if, regardless of modules,
standard layouts would have benefits in term of dependency management.

## Make the module name part of the file name

This is I think the approach that is the simplest, the saner and the easier to agree upon.

**A module `foo` would have to be declared by a file whose name is `foo.cppm`, a module `foo.bar` would have to be declared by a file whose name is `foo.bar.cppm`.**
And that would be it - it's quite simple.

This would solve the issue exposed above while being a rather small constraint.
It would make refactoring code easier and the dependency graph slightly less dynamic (Renaming a file is easier to track by a build system than just modifying the `export module foo` expression).

Given that the characters used by modules identifiers are a subset of what is supported by most of all build system, there would be a 1 to 1 correspondence between file name and module name.
The only thing we would have to agree upon is an extension. Which seems doable once we agree that this is a problem that needs solving.

I could argue there is precedence for that. after all, there is a 1 to one correspondence between the directive `#include 'foo.hpp'` and the file `foo.hpp`.

This scheme is actually implemented by `build2`. [The build2 documentation](https://build2.org/build2/doc/build2-build-system-manual.xhtml#cxx-modules-build) explains:

> To perform this resolution without a significant overhead, the implementation delays the extraction of the actual module name from module interface units (since not all available module interfaces are necessarily imported by all the translation units). Instead, the implementation tries to guess which interface unit implements each module being imported based on the interface file path. Or, more precisely, a two-step resolution process is performed: first a best match between the desired module name and the file path is sought and then the actual module name is extracted and the correctness of the initial guess is verified.

> The practical implication of this implementation detail is that our module interface files must embed a portion of a module name, or, more precisely, a sufficient amount of "module name tail" to unambiguously resolve all the modules used in a project. Note also that this guesswork is only performed for direct module interface prerequisites; for those that come from libraries the module names are known and are therefore matched exactly.

Unfortunately, `build2` module<->file mapping is fuzzy and as such more brittle.
The documentation argues that:

> While we could call our interface files hello.core.mxx and hello.extra.mxx, respectively, this doesn't look particularly good and may be contrary to the file naming scheme used in our project.

However, is this flexibility worth the added complexity? I really don't think so!

Enforcing the same, trivially implementable mapping also guarantees that all build system behave similarly.

Designing a C++ build system is hard.
Let's not make it harder.






































































