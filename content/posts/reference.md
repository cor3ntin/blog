---
title: "How I use references"
date: 2020-02-25T12:40:00+01:00
---

Following a blog post by [Herb Sutter](https://herbsutter.com/2020/02/23/references-simply/), let me tell you how and when I use references.

* If I do not need to mutate an input parameter, I will use a const reference,
unless I know that copying is cheaper (When in doubt use a reference).
* If I do need a copy of the parameter, I accept it by value and move it
* If I do need to mutate an input parameter, I will accept an input reference. But in many cases, I prefer to take the parameter by value and return a copy.
* I avoid out parameters. Returning by value is cheap, always preferable.
* I use references and const references to make local aliases.
* I avoid [rvalue references](https://cor3ntin.github.io/posts/move/)

A bit of code is better than a blob of text:

```cpp
void f(const Foo &); // input parameter
void f(Foo);         // input parameter, modified or sinked
void f(Foo &);       // input-output
Foo f();             // output
```

I try to never use pointers, at least in interfaces.
Pointers should be non-owning, but in practice, there is too much C and legacy code for this to be the case.
When I see a pointer, I am wary of ownership and lifetime. I always will be, because we cannot magically get rid of owning pointers.

These are general advices, and I recommend to follow them in in interfaces.
Be adventurous at your own risk else where and use good judgment!

## What if I need an optional reference?

Use a default parameter if you can, a pointer if you must.

`std::optional<T&>` would be an obvious answer, sadly it is something the committee
refuses to standardize mainly because we cannot agree on what assignment and comparison should do.
The answer is however quite simple: [**these operations should not be provided**](https://wg21.link/p1175r0).
I don't know why but many people seem compelled to provide operators for everything when it doesn't make sense or is otherwise confusing.
**When there is no good default, do not try to provide a default**

Here is how to support reference in optional without having to specialize it.

```cpp
template<class T>
  class optional {
  public:
    [[deprecated]] template<class U = T>
    requires std::semiregular<T>
    optional& operator=(U&&);
  };

template<class T, class U>
requires (std::regular<T> && std::regular<U>)
constexpr bool operator==(const optional<T>&, const optional<U>&);
```

This would be nothing new - we do that for views. Wrapper objects should never try to expose more regularity than the wrapped type.

We removed `span::operator==` and, unsurprisingly, exactly nobody is missing it.
Instead of endless, unsolvable debates of what a given operation should do, a better question is: is that operation useful? The answer here is no, look at usages.

And this is in line with Herb Sutter's argument that references are mostly useful as
return values and parameters.

## What about not_null, object_ptr, observer_ptr etc?

`optional<Foo&> opt{foo};` is valid by construction. As long as you never use pointers,
the only way to misuse `optional` is to deference it while it is not engaged.

Types constructed from a pointer give you twice as many opportunities to blow your foot off:
while referencing and while constructing. It's shifting a compile-time error to a runtime one...not
the right direction to move in!
Additionally, `optional` provides useful functions like `value_or`, [`or_else`, `transform`](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p0798r4.html#transform)...

## Are reference just pointers?

It's memory addresses all the way down but it doesn't matter how references are implemented. Because they are non-null and can't be rebound, they behave as aliases, tied to the lifetime of the aliased object.
