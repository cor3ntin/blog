---
title: "Towards Better C++ Modules - Part 2: Modules naming"
date: 2018-12-06
---

In case you have been napping, [in the last installment we talked about modules mapping](/posts/modules_mapping), and so now we must talk
about modules naming.

You see, modules have names and names are hard.
In this case, names serve to identify uniquely each module used through the program.

The end of this article proposes to govern module naming through an official WG21 standing document and I would love _your_ opinion.
But be patient!

# Dotting the identifiers

An important point to make is that modules names are composed of a sequence of identifiers separated by dots.
You might think dots have been granted some semantic meaning, the power of organizing the world hierarchically.
And you would be wrong.
Dots are dots. And nothing more.
They have no more meaning than what you would be inclined to ascribe to them.
And so, all modules are created equal. There are no submodules, no super module, no set or superset.

`foo.bar` and `foo.baz`, as far as The standard is concerned, are not related.
`foo` and `foo.bar` aren't either.
Importing  `foo` does especially not import the names of `foo.bar`.
You can't go wild and `import foo.*;` because there is no such thing.

What you can do, however, like all respectable spy agencies, is start an `export import` scheme.
For example, given a module `foo.bar` (declared in `foo.bar.cppm`), you can further have a module
`foo` in a file `foo.cppm`

```cpp
export module foo;
export import foo.bar;
```

Doing so makes all names exported by `foo.bar` visible to `foo` and all other translation units importing foo,
thereby creating a hierarchy.
But `foo` could equally `export import` `bar` or `foo_bar`;

# Say my name.

You very enthusiastically decide to put all your core business logic in `core.cppm` and some useful bits in `utils.cppm`.

Sometimes later, you start using a third party library which has 2 modules: `core.cppm` where lies the core of the library, and the aptly named `utils.cppm` containing
some useful bits of that library.

There was no collusion here, just a violent collision abruptly ending the compilation in a terrible slam.
Obviously, you did nothing wrong. You were first to claim `core.cppm` for yourself.
If only one can use that name, it should be you. `core` is a great name and it's yours. For now and ever.

Others disagree, conflict arises, we are in a bit of a pickle.

# A bit of Elvish

Where would you find `namespace aragorn`? Well, in the `strider` module, as one would expect. In turn, that module is logically located
in the `esstr` directory (Originally called `elessar_telcontar`, but that proved problematic in regard to `MAX_PATH` as windows developers didn't quite care for old Entish).
The whole thing is part of the `Longshanks` project, that you will find on `github.com/Tolkien/D\&#250;nadan`.

It is fortunate indeed that linguists are not C++ developers.

And while most reasonable projects are not as intricate as _The Silmarillion_, the fact remains that a lot of entities have to be created and named:
Libraries, modules, directories, files, namespaces...

[In my previous article on module mapping](/posts/modules_mapping), I talked about the benefits of giving modules and files the same names.
One thing I didn't mention is that names are hard to find and harder to remember. Naming things uniformly make for an easier to read codebase.

Liberated of the pressure of naming files ([What can I say except you're welcome?](https://www.youtube.com/watch?v=79DijItQXMM)), let's focus on libraries and namespaces.

If a module is a collection of names, then a namespace is a named collection of names and a library is a named collection of names with a ribbon.
Of course, a module can open several namespaces, a namespace can spread over several modules, and a library can be composed of several namespaces and modules.
There are header-only libraries and there will be module-interfaces only libraries.

Maurits Escher was 25 years old when John Venn died. Did they met?

{{< figure src="doll.jpg" title="Modules, libraries and namespaces" attrlink="http://angelsbarcelona.com/en/artists/jaime-pitarch/projects/chernobyl/192" attr="Jaime Pitarch" >}}


### A daily reminder

**A module does not a namespace make**.

Modules are not namespaces and they do not introduce a namespace or provide any kind of namespacing or prefixing or anything of the sort to the names they export.
Because modules are close-ended and namespaces can be reopened, I do not believe this could possibly be changed or improve upon. Sad Face Emoji

This was your daily reminder that **a module does not a namespace make**.

## Namespaces and Libraries

We understand that putting names in the global namespace is bad.
We also think that ADL makes namespaces terrible.

That doesn't leave us a lot of places to put names.

Being reasonable, we agree that each library should have one top-level namespace containing all its names and maybe avoid nested namespaces.

We also know that putting names in other people namespaces will lead to breakage when they themselves introduce the same names and as such,
opening other people's namespaces is frowned upon.

**Top level namespaces, therefore, do not denote a cohesive set of names but rather signal ownership**.

Libraries also signal ownership. Even if there is a logical unity (a library often provides a cohesive set of features),
the defining property of libraries is to have an owner, an entity that provides or maintains that library.

And so, namespaces and libraries provide the same functionality: Signaling Ownership.
Being two side of the same coins, maybe namespaces and libraries should share the same names?


Did I mention Naming is hard? [`Argh!`](https://github.com/adishavit/argh)

[`Loki`](http://loki-lib.sourceforge.net/), a [`crow`](https://github.com/ipkn/crow) [`CUTE`](http://cute-test.com/) as a [`botan`](http://botan.randombit.net/)  [`wangle`d](https://github.com/facebook/wangle) a [`pistache`](http://pistache.io/) while I drank this [`Tonic`](https://github.com/TonicAudio/Tonic) [`Acid`](https://github.com/Equilibrium-Games/Acid) [`Yuzu`](https://github.com/yuzu-emu/yuzu) [`juce`](https://github.com/julianstorer/JUCE) giving me a [`boost`](https://github.com/boostorg).
Is  [`json`](https://github.com/nlohmann/json) a good name? [`Nope`](https://github.com/riolet/nope.c)! [`Hoard`](https://github.com/emeryberger/Hoard) of projects are already called like that, it would be [`reckless`](https://github.com/mattiasflodin/reckless) [`folly`](https://github.com/facebook/folly).

(If you can make a fun sentence composed of C++ project names, I will retweet it !)

Library and project names are usually creative.
Yet, they need to be unique, and at the same time, on the shorter side if at all possible.


But how can a name be short and creative while remaining creative and meaningful?


# Naming through the ages

## Java

Java packages offer the same features as C++ modules and namespaces combined.
The java documentation states

> Companies use their reversed Internet domain name to begin their package names—for example, com.example.mypackage for a package named mypackage created by a programmer at example.com.

> Name collisions that occur within a single company need to be handled by convention within that company, perhaps by including the region or the project name after the company name (for example, com.example.region.mypackage).

> Packages in the Java language itself begin with java. or javax.


Java is almost 25 years old and yet wise enough to propose a naming scheme that guarantees uniqueness and signal ownership

## C&sharp;
`C#` has assemblies (≈ libraries) and namespaces and does not need modules.

It provides an [impressively detailed guideline](https://docs.microsoft.com/en-us/dotnet/standard/design-guidelines/names-of-namespaces),
for the naming of namespaces, which is summarized as:
    `<Company>.(<Product>|<Technology>)[.<Feature>][.<Subnamespace>]`

> ✓ DO prefix namespace names with a company name to prevent namespaces from different companies from having the same name.

> ✓ DO use a stable, version-independent product name at the second level of a namespace name.

I'm not familiar with `C#`, but I assume it doesn't suffer from the use of nested namespaces.
Most importantly, `<Company>.<Product>`, should be unique, and be immutable.

## Go

Go realized that packages are resources that need to be uniquely identified and so Go packages can be
imported through an URL.

It also offers some [insights about good packages names](https://blog.golang.org/package-names). Apparently, `util` is
not a good package name. Who would have thought?


## Rust and Javascript

Yes, I dare to bundle these two together, I double dare.

Rust has crates which are the combination of C++ libraries and modules. Rust also has modules, which are namespaces.
The Javascript ecosystem has packages (libraries) made of modules behaving like namespaces, often implemented as functions.
Confused?

Fortunately, both Rust and Javascript have official or de-facto centralized package managers (cargo and npm respectively).
That centralized package manager guarantees the uniqueness of package name, using a simple scheme:
First arrived, first served.

NPM offers the possibility to prefix a package name by an organization name (`google/foo_bar`), while cargo does not.
This is, as it turns out, a recurring topic in these communities.


# The library that owns itself

Let say you want to use `Qt`, a great library that does 2d graphics, audio and even encrypted networking.
Qt was developed by Trolltech in the early 90s.
So, Trolltech owns Qt, and because company names are reasonably unique, `trolltech.qt` is unique and would rename unique forever.

In 2008, Trolltech was bought by Nokia. Then Nokia was bought by Microsoft and Qt was bought by Digia who then spawned The Qt Company.
Meanwhile, Qt is also an open source project maintained by the `Qt Project` who exists in part thanks to the `KDE Free Qt Foundation`.
In 2012, some people decide to create a new project called CopperSpice out of a fork of Qt.

You probably know `Catch`. It's a great testing framework. But do you know Phil Nash, The great guy who created Catch?
Since then, lots of people have contributed to Catch, which is developed at [github.com/catchorg/catch2](github.com/catchorg/catch2).
So who maintains `Catch`? The `Catch` maintainers, obviously!

In fact, most open source libraries are owned by their maintainers, which means they are own by everyone and no one at the same time.
And so, should 'Catch' be refered to as `catch` `philnash.catch` or `catch.catch` ? ([oups, `catch` is a keyword](https://godbolt.org/z/c2LrCM)!)

More importantly, projects can be forked.

If Microsoft forks Google's fork of Webkit, is it still Google's? Should it be called `google.blink` or `microsoft.blink`?
Or just [`apple.wtf`](https://chromium.googlesource.com/chromium/src.git/+/62.0.3178.1/third_party/WebKit/Source/platform/wtf/README.md) ?

If Opera were to buy both Google and Microsoft, and all the modules and top-level namespaces names are different, would they be able to ever merge these 2 projects back together?

These are real concerns (watch out Microsoft!), because names, like diamonds, are forever.
Top level namespaces and module names even more so.

Like top level namespaces, module names will be very invasive and spread like _The Great Plague_, or _The GPL_.
Both modules and namespaces can have aliases (with `export import` for modules), but they can never disappear.

If you look at old java projects, `import` declarations show the geological record of a bygone era when Sun shined on the ecosystem.

It's not just a matter of API either, modules names can be made part of the ABI. They can't be renamed, _ever_.


# Making sure the future is backward compatible

We don't have a dependencies manager of meaningful scale.
But, unicity of names is central to any such tool. `vcpkg` for example use project names to identify packages and requires names to be unique.
Having uniquely addressable packages offer many great benefits and opportunity for amazing tooling.
Having further consistency between project names, module names and library names ensure there are no name collisions and that all libraries can be easily
used in the same project.

Imagine a tool that download boosts when you type `import boost.system.error;`  or one that inserts `import folly;` when you type `folly::`.

# A call for a standing document

While _The C++ Standard_ cannot enforce good names, a great many languages provide guidelines for package/namespace/modules/etc
naming and I think it is important C++ do the same.

The goal is not to enforce unique names (because it is not possible), or to overly constrain naming scheme, but to make sure people do not name their projects in a way
that would hinder the development of a larger ecosystem.

The C++ Core Guidelines might be another area to explore,
but they are less official and we can only reap the benefits of consistent naming if everybody follows the same rules.


## Rough Draft:

* **Prefix module names with an entity and/or a project name to prevent modules from different companies, entities and projects of declaring the same module names.**
* **Exported top-level namespaces should have a name identic to the project name used as part of the name of the module(s) from which it is exported.**
* **Do not export multiple top-level namespaces**
* **Do not export entities in the global namespace outside of the global module fragment.**
* **Organize modules hierarchically.** For example, if both modules `example.foo` and `example.foo.bar` exist as part of the public API of `example`, `example.foo` should reexport `example.foo.bar`
* **Avoid common names such as `util` and `core` for module name prefix and top-level namespace names.**
* **Use lower-case module names**
* **Do not use characters outside of the basic source character set in module name identifiers.**


# Conclusion

Modules might give the C++ community the rare opportunity to federate the ecosystem under a common set of rules.\\
This set of rules would permit the emergence of more modern module-oriented dependencies managers and tools.

As modules cannot be renamed, these rules would have to be published along with the same C++ version that introduces modules as a language feature.

What do you think?





































