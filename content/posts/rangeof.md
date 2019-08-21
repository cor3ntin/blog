---
title: "RangeOf: A better span"
date: 2018-11-25T11:50:51+01:00
---

[I don't like span](/posts/span).

Since that article was posted, the committee improved span quite a bit by removing `operator==` and making it's `size()` consistent
with that of `vector` after a lot of discussions. And I mean _a lot_.

# What is span: 30 seconds refresher

If you have _N_ `T` laid out contiguously in memory, you can build a `span<T>` over them. Span being a value type you can move it around,
copy it and so forth. But since the `span` doesn't own its elements, the underlying data must outlive it.

## Some issues

 * Span is a value type but being non-owning, it should be seen as a pointer, maybe even be called `spanning_pointer`
 * Being non-owning, it's a view in the Range terminology. Its constness is _shallow_. that means that you can modify the underlying element of a `const span`
 * Being a pointer over contiguous memory, you can only make a span over a C array, a `std::vector`, `boost::vector` and so forth.

## What is span good for?

 * It allows manipulating sub-ranges without having to copy data around
 * It allows using contiguous containers homogeneously without having to care about their type and it does that without incurring a lot of template instantiation.

# A better solution

Consider the following function

```cpp
template <typename T>
void f(const std::span<const T> & r);
```

With Ranges and the terse syntax merged into C++20, we can instead write something like that

```cpp
void f(const std::ContiguousRange auto & r);
```

From the caller perspective, both these function will behave identically and the implementations will be very similar too.
Except the one taking a range has a much easier to understand ownership model

If called with more that one type of containers the `span` version will be instantiated only once per element type, whether
the ContiguousRange will be instantiated per range type.
Keep that in mind if you work with memory constrained platforms.
But in general, I think [we should try to move away from the Header/source file separation model](/posts/translation_units) so we can make full use of `constexpr` functions,
generic code and the ability of the compiler to do code inlining.

Anyway, How can you specify that you want a range of a specific type?
With span, it's quite straightforward:

```cpp
void f(const std::span<const int> & r);
```

With ranges, it would look like that:

```cpp
template <std::ContiguousRange R>
requires std::is_same_v<std::ranges::iter_value_t<std::ranges::iterator_t<R>>, int>
void f(const R & r);
```
There we are done. Easy, right?
With a bit of luck that might be simplifiable further by C++20:

```cpp
template <std::ContiguousRange R>
requires std::is_same_v<std::ranges::range_value_t<R>, int>
void f(const R & r);
```

And it's easy to use using `ranges::subrange`:

```cpp
int main() {
    auto v = std::vector<int>(42, 0);
    f(v);
    f(v | ranges::view::take(5));
    f(ranges::subrange(v.begin() + 1, v.begin() + 3));
}
```

Simple, right? Hum...it still quite verbose, isn't it?

I think it would be nice to be able to write

```cpp
void f(const std::ContiguousRangeOf<int> auto & r);
```

Fortunately, concepts can be parametrized, so this can be easily defined:

```cpp
namespace std {
template <typename R, typename T>
concept ContiguousRangeOf = ContiguousRange<R> &&
    std::is_same_v<ranges::iter_value_t<ranges::iterator_t<R>>, T>;
}
```

(The first template parameter is the type the concept is applied to)

Besides being easier to understand that span as far as ownership goes and not introducing new
types, it's also generalizable to  all kind of ranges, not just contiguous ones, and as such can be used
with all kind of containers and views.

```cpp
namespace std {
template <typename R, typename T>
concept RangeOf = Range<R> &&
    std::is_same_v<ranges::iter_value_t<ranges::iterator_t<R>>, T>;

template <typename R, typename T>
concept ForwardRangeOf = ForwardRange<R> &&
    std::is_same_v<ranges::iter_value_t<ranges::iterator_t<R>>, T>;

template <typename R, typename T>
concept BidirectionalRangeOf = BidirectionalRange<R> &&
    std::is_same_v<ranges::iter_value_t<ranges::iterator_t<R>>, T>;

template <typename R, typename T>
concept RandomAccessRangeOf = RandomAccessRange<R> &&
    std::is_same_v<ranges::iter_value_t<ranges::iterator_t<R>>, T>;

template <typename R, typename T>
concept ContiguousRangeOf = ContiguousRange<R> &&
    std::is_same_v<ranges::iter_value_t<ranges::iterator_t<R>>, T>;
}
```

Now, we can for example write:

```cpp
void f(const std::RangeOf<std::string> auto & r);
```

## Concept template

Unfortunately, concepts cannot be used as template parameter (_yet_ ?),
so it is not possible for example to define a `std::RangeOf<Number>`.
I hope this limitation will be lifted by C+23.


# Conclusion

While `span` has its place, notably on embedded platforms, shying away from templates and concepts in the hope of slightly faster compile times forces us to deal with types that are easy to misuse and fit poorly in the C++
type system.

Instead, ranges and the terse syntax give us less surprising tools to express their
same idea in a simpler, better-understood manner.
And we can add some sugar coating so that simple ideas can be expressed without `requires` clauses.

Do you think `RangesOf` would be useful enough to be added to the standard library?
In the meantime, you can [play with it on Compiler Explorer](https://gcc.godbolt.org/z/M396_s).