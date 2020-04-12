---
title: "A few experimental features for C++"
date: 2020-03-06T16:00:00+01:00
---

In this article, I present a few language features that I am hoping to see
in C++23 and which I have deployed to Compiler Explorer.

**Please note that these features are not part of a working draft and they have not been presented
to the C++ committee yet, so it is impossible to comment on whether any of them might land in 23 or not!**

## Auto Non-Static Data Members Initializers

A while back I presented [auto non-static data members initializers](/posts/auto_nsdmi).
At the time, it was based on a clang 7 fork.
Because this is still a feature I would like to see in a future C++ version, I rebased it on top
of Clang 11, which was a bit complicated because of LLVM's migration to a monorepo (but I am very glad they did that migration!).

You can play with it on compiler explorer here:

{{< ce compiler="autonsdmi">}}
#include <vector>
struct s {
    auto v1 = std::vector{3, 1, 4, 1, 5};
    std::vector<int> v2 = std::vector{3, 1, 4, 1, 5};
};
{{< /ce >}}

There is no proposal for this feature yet.
I am hoping to convince people to work on it!

## Multidimensional subscript operator

The idea is very simple: It's about allowing multiple arguments
in subscript expressions:

```cpp
struct image {
    pixel operator[](size_t x, size_t y) const;
};
/*...*/
pixel x = my_image[42, 42];
```

In C++20, [we deprecated](http://eel.is/c++draft/depr.comma.subscript) `,` in subscript expressions:
A warning is already implemented in GCC and Clang.
MSVC warns about surprising syntax but doesn't mention deprecation yet.

```cpp
int main() {
    int array[2] = {3, 4};
    //warning: top-level comma expression in array subscript is deprecated
    //(equivalent to array[(0, 1)], equivalent to array[1])
    return array[0, 1];
}
```

In C++23, we are hoping to reuse the syntax so that subscript expressions can accept
any non-null numbers of arguments.
This is important to make the interface of [mdspan](https://wg21.link/p0009) and [mdarray](https://wg21.link/p1684r0) more intuitive.
These classes currently overload the call operator, which encourages wild operator overloading.
Many domains could benefit from this feature, including linear algebra, image manipulation, audio, etc.

{{< ce_fragment compiler="autonsdmi"  >}}
{{< ce_hidden >}}
#include <boost/multi_array.hpp>
#include <type_traits>
#include <vector>
{{< /ce_hidden >}}{{< ce_code >}}
template <typename T, std::size_t N>
class mdarray : protected boost::multi_array<T, N> {
public: {{< /ce_code >}}{{< ce_hidden >}}
    using base = boost::multi_array<T, N>;
    using base::base;

    template <typename... Idx>
    requires (sizeof...(Idx) == N
             &&  (std::is_nothrow_convertible_v<Idx, std::size_t> && ...))
    mdarray(Idx... idx)
    : base( boost::array<typename base::index, N>({idx...})) {};


    {{< /ce_hidden >}}{{< ce_code >}}    // variadic operator []
    template <typename... Idx>
    requires (sizeof...(Idx) == N
      &&  (std::is_nothrow_convertible_v<Idx, std::size_t> && ...))
    T & operator[](Idx... idx) {
        boost::array<typename base::index, N> id({idx...});
        return this->operator()(id);
    }
};

int main() {
    mdarray<int, 2> arr(2, 2);
    arr[1, 1] = 42;
    return arr[1, 1];
}
{{< /ce_code >}}
{{< /ce_fragment >}}

**This feature is described in [P2128R0 - Multidimensional subscript operator](https://isocpp.org/files/papers/P2128R0.pdf)
and will be presented to the C++ committee at a future meeting.**

## A placeholder with no name

Naming is hard. It's even harder to name variables you do not care about.
There are a few cases where variables names do not matter in C++, for example:

* Any kind of RAII guard such as a mutex lock that is never unlocked manually

```cpp
std::unique_lock my_lock(m);
```

* Some values in structured bindings

```cpp
auto [result, i_dont_care] = my_map.insert(42);
```

* Variables stored in lambda captures to extend their lifetime

```cpp
std::unique_ptr<T> ptr = /*...*/;
auto& field1 = ptr->field1;
auto& field2 = ptr->field2
[really_do_not_care=std::move(ptr), &field1=field1, &field2=field2](){...};
```

(Example stolen from [P1110](https://wg21.link/p1110))

* Global variables used for self-registration and other side effects

This last example is often wrapped in macros which try to create
unique identifiers with `__LINE__ ` and `__COUNTER__` at global scope.

```cpp
auto CONCAT(__register_foobar_, __LINE__, __COUNTER__) = register_type<Foo>("Foo");
```

Many languages use the `_` identifier as a magic identifier meaning "I don't care about the name",
including Go, Rust, Scala, Haskell. Python similarly uses `_` the same way by convention.

Unfortunately, `_` is not currently reserved in C++ (except in the global namespace) and it is used by a few frameworks such as GoogleTest, also to mean "I don't care".

[P1110](https://wg21.link/p1110) considers a few alternative syntaxes such as `__`, `?` and `??`.
But I think `_` is the most elegant identifier for that purpose. We should strive to use it, both for readability and consistency across languages, which I think matters when possible.

As [P1469 - Disallow _ Usage in C++20 for Pattern Matching in C++23](https://wg21.link/p1469) notes,

> Why is `_` so important when `?` is available? Languages with pattern matching almost universally use `_` as a
> wildcard pattern and popular libraries in C++ (like Google Test) do the same. It would be awkward and
> somewhat embarrassing if C++ were to not use such a ubiquitous token. Furthermore, because `_` has so
> much existing widespread use, we expect people to use `_` anyway, and accidentally bind the `_` identifier.

Fortunately, there is a way to be able to use `_` as a placeholder identifier, while not breaking the few libraries
using it as a namespace-scope variable identifier:

We can make `_` magic only if a `_` already exists in scope. Aka, it would become maginc only on second use.
This solution works very well for nameless captures, structured bindings and RAII guards alike,
while carefully avoiding breaking any existing code.

{{< ce compiler="autonsdmi">}}
#include <map>
int main() {
    std::map<int, int> m;
    auto [it, _] = m.emplace(0, 42);
    auto [_, value] = *it;
    return value;
}
{{< /ce >}}

Of course, one other use case for `_` is to silent unused variables, as if they were marked `[[maybe_unused]]`:

{{< ce compiler="autonsdmi">}}
[[nodiscard]]
int f() {
    return 42;
}

int main() {
    auto _ = f();
    // result discarded
    f();
    // unused variable
    auto foo = f();
}
{{< /ce >}}


We can deprecate a few usages of `_` as an identifier, notably for types, concepts, modules, aliases, etc.

The downside of this approach is that in some cases it might be a little bit confusing
to know whether a variable introduced by `_` is anonymous or not. But these cases can be diagnosed quite well.

{{< ce compiler="autonsdmi">}}
struct raii {
    raii();
};

int main() {
    int _ = 42;
    raii _;
    return _; // warning: Refering to a variable named '_'
              // while anonymous variables are in scope
}
{{< /ce >}}

Because of linkage and ODR concerns, `_` as a magic blank identifier cannot be used at namespace scope.
We could, however, allow it in modules units if they are not exported, which would be very useful
to declare variables that are only used for the side effects of their initialization.

{{< ce compiler="autonsdmi">}}
export module m;

int _ = 42;
int _ = 47;

{{< /ce >}}

Please note that this is not fully implemented yet, as these variables would need
special mangling.

EWG-I seemed interested in the general idea of placeholder names such as described
in [P1110](https://wg21.link/p1110r0).
There is however no proposal for the specific behavior described here yet.
I will see if I can collaborate with a few papers for Varna.

## That's all folks

These are small features, but they can help make the language a bit
more intuitive.

Let me know what you think!

A huge thanks to Matt Godbolt and the rest of the Compiler Explorer team.