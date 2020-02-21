---
title: "move, even more simply"
date: 2020-02-18T13:45:26+01:00
---

`std::move` doesn't move.

It casts to an rvalue-reference, which is a type of reference that can
be passed to a move constructor or assignment operator, if one exists.

```cpp
template <typename T>
decltype(auto) move(T&& a) {
    return static_cast<std::remove_reference_t<T>&&>(a);
}
```

Some expressions will be converted to rvalue-references automatically,
when the compiler is certain that the value is expiring (will not be reused).

This is the case for temporaries or non-reference objects returned from functions.

In many cases, you still need to use `std::move` explicitly: C++ compilers will for example never check whether an object might be reused later in the function.

## So, if std::move doesn't move, is it a bad name?

No, because it shows the intent of `move`. The important point is that in the general case, by the end of the expression
or statement in which `std::move` appears, the object might have been moved-from.

[Herb Sutter is right](https://herbsutter.com/2020/02/17/move-simply/): move constructors, move assignment operators and
rvalue-reference qualified functions are just regular non-const functions.

But, because in the 99% cases the moved-from object will be swiftly destroyed after being moved-from, a class may decide to not restore all of the invariants in these functions, for performance reasons.

And so, in the absence of documentation that states and guarantees a known and well-behaved moved-from state,
it is best to assume that the only valid operations on moved-from objects are assignment and destruction.

It is also best to assume that an object that **may** have been moved-from has been moved-from.
Don't play with fire.

Should types offer stronger guarantees? Maybe, but that ship has sailed and in any case, it goes against the "don't pay for what you don't use" mantra as very few moved-from objects are ever reused.

Does the standard library offer stronger guarantees? Sometimes, but not always and often under-documented.

## This is still too complicated

**In the absence of other information, do not do anything to an object on which `std::move` has been called, except assignment operator and destructor.**

C++ move is not destructive but it might as well be.