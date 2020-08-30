---
title: "Non-terminal variadic template parameters"
date: "2020-06-16T15:00:00"
---

A few months ago I [presented a few features](/posts/qol23) that I hope will be considered for C++23.
I have since then submitted papers for [multi-dimensional subscript expressions](https://wg21.link/p2128r1) and
['_` as a variable placeholder](https://wg21.link/p2169).

In this article I want to talk about another improvement I would like to see in the next C++ version:
Non-trailing variadic template parameters.

Indeed, while parameter packs can appear before the last function parameter, they do not get
properly deduced when they do.

```cpp
template <typename... A, typename B>
void f(A...a, B b); // OK

f(0); // Compiler is confused
```

Note that, they can appear in a different order in the template head, that's perfectly fine:

```cpp
template <typename B, typename... A> // OK
template <typename... A, typename B> // also OK
```

Unfortunately, this limitation is an issue in many instances.

For example, many people are surprised that `std::visit` accept
the visitor argument first, and this is because it accepts a variadic number
of variants:

```cpp
template <class Visitor, class... Variants>
constexpr auto visit( Visitor&& vis, Variants&&... vars );
```

The following would make a better interface (easier to read, symmetric with `std::transform`, etc) but would not be viable because
of the current deduction rules.

```cpp
constexpr auto visit(Variants&&... vars, Visitor&& vis);
```

### source_location

In C++20, we added `source_location`, such that `source_location::current()` returns the location
(filename, line number etc) of where that call is made.

Typically, you would use it as default parameter of a log function:

```cpp
void log(std::string view, std::source_location loc = std::source_location::current());
```

But what if your log function accepts a variadic number of arguments?
For example,  [spdlog](https://github.com/gabime/spdlog) uses [fmt](https://github.com/fmtlib/fmt),
such that its log function looks like this:

```cpp
void log(std::string view, auto... args);
log("Hello {}", "World");
```

We would probably want to write it as follow, but we can't
```cpp
void log(std::string_view, auto... args,
         std::source_location loc = std::source_location::current());
```

A clever [stackoverflow](https://stackoverflow.com/a/57548488) contributor suggested that it can be achieved
through deduction guide as follow:

```cpp
template <typename... T>
struct log {
    log(std::string_view, T&&...,
        std::source_location loc = std::source_location::current());
};

template <typename... T>
log(std::string_view, T&&...) -> log<T...>;
```

This works by avoiding having trailing default parameters during the initial overload resolution of the call to log.
It is, however, still the object of an [active issue](http://www.open-std.org/jtc1/sc22/wg21/docs/cwg_active.html#1609),
but all compilers seem to accept that code.

## apply_last

A while back I wanted to use `std::apply`, but I needed to handle the last tuple element
differently from the rest, which is a common pattern.
Here is what I came up with.

```cpp
template <class F, class Tuple>
constexpr decltype(auto) apply_last(F &&f, const Tuple &t) {
    return [&]<auto... I>(std::index_sequence<I...>) {
        return f(std::get<std::tuple_size_v<std::remove_reference_t<Tuple>> - 1>(t),
                std::get<I>(t)...);
    }(std::make_index_sequence<std::tuple_size_v<std::remove_cvref_t<Tuple>> -1>{});
}

void f() {
    apply_last([](auto && last, auto&&... first) {
        assert(last == 3);
    }, std::tuple{1, 2, 3});
}
```

There is some amount of complexity here.
Most notably, the `last` parameter appears first on the lambda, on the caller side.

It would be a lot more intuitive to be able to write:

{{< ce compiler="autonsdmi">}}
#include <tuple>
#include <cassert>
#include <utility>
void f() {
    std::apply([](auto&&..., auto && last) {
        assert(last == 3);
    }, std::tuple{1, 2, 3});
}
{{< /ce >}}

And you can click the compiler explorer icon to test this
experimental code!

In fact this feature was already proposed in 2016 in [P0478R0](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0478r0.pdf) by Sy Brand, Bruno Manganelli and Michael Wong.
Bruno Manganelli is notably responsible for the clang implementation.
I merely did the not-so-fun work of porting it from clang 7 to clang 11, so that we can all play with it.

When it was first presented, WG21 felt the motivation wasn't strong enough.
I think there is more motivation now, and with the help of the original authors,
I am hoping we can present that paper again.
I expect the wording will be challenging though.

But frankly, given that their appearance is legal and it falls on its
head during argument type deduction, and given the implementation
divergence between GCC and MSVC, I am enclined to think this is
a bug fix as much as it is a feature.


## How does it work?

It is hard to argue that overloading rules in C++ are not complicated.
But, in theory that feature is actually quite intuitive

lets say you have

```cpp
f(auto a, auto... b, auto c, auto d);
f(1, 2, 3, 4, 5);
```

We know that we have 5 arguments and `f` has 3
non-variadic parameters (`a`, `c` and `d`).

So for the call to `f` to be valid, we can synthesize
the following overload

```cpp
f(auto a, auto b1, auto b2, auto c, auto d);
```

After that argument type resolution and overload
resolution applies as it would to any non-variadic function.

The story is a bit less clear cut when there are default arguments.
Our implementation will eagerly make the pack as large as possible

```cpp
f(auto a, auto... b, auto c, auto d = 1);
f(1, 2, 3, 4, 5);
// => f(auto a, auto b1, auto b2, auto b3, auto c, auto d = 1);
```

But a better strategy would be to make this scenario ambiguous
which is less error-prone and more consistent.

That doesn't mean that variadic parameters and default parameters cannot co-exist.
Let's go back to the `source_location` use case:

```cpp
void log(auto... args, source_location = source_location::current());
```

A call to `log(1, 2)` is unambiguous.
after argument type deduction, the following function can be imagined to exist: `void log(int, source_location)` and `void log(int, int, source_location = /*...*/)`.
Overload resolution is then not ambiguous, the program is well-formed.

But what about `log(source_location{})`?

We can imagine both these to be synthetized:

```cpp
void log(source_location = source_location::current());
void log(source_location, source_location = source_location::current());
```

Which during overload resolution would be considered ambiguous, and therefore ill-formed.
But what if you want to write a function that forward all of it's argument while still having default parameter of its own?
Maybe `log(source_location{})` is a perfectly reasonable thing to do!

This can be solved by a tag on the callee side, without further language modification

```cpp
struct my_end_of_parameters_tag_t{};
void log(auto... args,
         my_end_of_parameters_tag_t = {}, source_location = source_location::current());
```

now, the call `log(source_location{})` can synthetize

```cpp
void log(my_end_of_parameters_tag_t, source_location = source_location::current());
void log(source_location,
        my_end_of_parameters_tag_t = {},
        source_location = source_location::current());
```

which during overload resolution correctly selects `log(source_location, my_end_of_parameters_tag_t, source_location)`

The general idea is provide to overload resolution a choice of two synthetized functions, one which assume that
an argument was provided for the first defaulted parameter, and one which was not.
Then overload resolution can do its thing without being modified.
The handling of interaction between variadic parameters and defaulted parameters is some thing that still needs to
be refined.


With these rules, a function would still be limited to one parameter pack,
but this parameter pack would be allowed to appear anywhere in the function
parameter list, and argument type and overload resolution would behave in
a consistent and standard manner.

There are a few other proposals to improve the usability of parameter packs
[Generalized pack declaration and usage ](https://wg21.link/p1858r2),
[Simplified structured bindings protocol with pack aliases](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2120r0.html)

## That's all folks

Let me know what you think!
Please play with this feature on [Compiler-Explorer](https://compiler-explorer.com/z/CsNBJD) and let me know about
the use cases you would have for such feature!

As often, A huge thanks to Matt Godbolt and the rest of the Compiler Explorer team.