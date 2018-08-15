---
title: "The case for Auto Non-Static Data Member Initializers"
date: 2018-08-12T11:27:00+02:00
---

In this article, we talk about Auto Non-Static Data Member Initializers in C++.
All code snippet can be tested on Compiler Explorer thanks to Matt Godbolt and the CE team.
The clang patch to enable this feature was authored by Faisal Vali 5 years ago,
but I have crudely rebased it on top of clang trunk (~ 7.0).

In fact, the main motivation for this article is to put this feature in the hand of people
to prove that it works and that it would be a great addition to the standard.

Having the capacity to test proposed features on Compiler Explorer is a great way to better
understand a feature and its corner case. **So I encourage you to play with the code snippets**.

But first thing first.

## What are Auto Non Static Data Member Initializers (<abbr title="Non Static Data Member Initializers">NSDMI</abbr>) ?


### Data Member Initializers
In C++, you can introduce a default value for a member variable, that will be used to initiate a variable if you don't initialize explicitly,
either in a constructor member initializer list or by using an aggregate initialization.

{{< ce >}}
int main() {
    struct S {
        int a = 42;
    };
    S s;
    return s.a;
}
{{< /ce >}}

This is called *Data Member Initializers*.
The initializer is only evaluated if the member isn't initialized explicitly.
For example, in the following example, `main` returns 0;

{{< ce >}}
int ret = 0;
int main () {
    struct {
        int x = ++ret;
    } x = {0};
    return ret;
}
{{< /ce >}}

### Static Data Member Initializers

In a similar fashion, static members can have an initializer, although, the rules are a bit different.
First, a static data member initializer is always evaluated and supersedes out-of-class definition.

The following code fails because we try to define `s::foo` twice:
{{< ce >}}
struct s {
    static const int foo = 42;
};
int s::foo = 42;
{{< /ce >}}

Only static data members that represent a literal value can have a data member initializer. This is because
otherwise, that static member needs to have linkage (be addressable at runtime, if you will) and as such only be defined
in the whole program. Otherwise, you would run into ODR violations. *gasp*.

### Auto Static Data Member Initializers

Static data members that have a *data member initializer* can be declared with auto.

{{< ce >}}
struct s {
    static const auto foo = 42;
};
{{< /ce >}}
In this case, `foo` is deduced to be of type `int` and it works exactly the same as any declaration of a variable with `auto`:
The right-hand side expression is evaluated and its type determines the type of the variable, in this case, the static data member.

### Auto Non-Static Data Member Initializers

With all those pieces, we can now see what an NSDMI is, simply a class or struct data member with an initializer, whose type is deduced.

{{< ce compiler="autonsdmi" >}}
struct s {
    auto foo = 42;
};
{{< /ce >}}

However, this won't compile: the standard forbids it.

## The case for auto NSDM

So, *Auto Non-Static Data Member Initializers* aren't actually a thing neither in C++17 or the upcoming C++20. It was last proposed in 2008,
and haven't generated a lot of discussions since - This blog post attempts to address that!

So, should the above code be valid? I definitively think so.
The argument really is... why not?

### Always Auto? Not quite.

That may sound like a poor argument, but data members are the only entity that can not be declared with `auto`.
`auto` can declare any kind of variables in all kind of contexts, but this one. And that kind of exception defies expectations.
Users might try to use them naturally, wonder why they don't work and then you would have to come up with a good explanation.

### Expressiveness of auto

The reason why you may want to use auto NSDMI is the same you would use `auto` in any other context. I think the strongest showcase at the moment would type deduction

{{< ce compiler="autonsdmi">}}
#include <vector>
struct s {
    auto v1 = std::vector{3, 1, 4, 1, 5};
    std::vector<int> v2 = std::vector{3, 1, 4, 1, 5};
};
{{< /ce >}}

`make_unique` and `make_shared` would also make good candidates, along with all `make_` functions

{{< ce compiler="autonsdmi">}}
#include <memory>
struct s {
    auto ptr = std::make_shared<Foo>();
    std::shared_ptr<Foo> ptr2 = std::make_shared<Foo>();
};
{{< /ce >}}


Literals can also make good candidates, however, they require a `using namespace` which you should avoid doing in headers. Which is more a problem with literals
and the inability to do using namespace at class-scope.

{{< ce compiler="autonsdmi">}}
#include <chrono>
using namespace std::chrono_literals;
struct doomsday_clock {
    auto to_midnight = 2min;
};
{{< /ce >}}

### It already works
As noted in [N2713 - Allow auto for non-static data members - 2008](https://wg21.link/n2713), almost anything that can be expressed by `auto` can be expressed with `decltype`

{{< ce >}}
struct s {
    decltype(42) foo = 42;
};
{{< /ce >}}

In fact, we can devise a macro ( please, don't try this at home )

{{< ce >}}
#define AUTO(var, expr) decltype(expr) var = (expr)
struct s {
    AUTO(foo, 42);
};
{{< /ce >}}

And, if it works with a less convenient syntax, why not make people life easier?

### Lambda data members

There is one thing that cannot be achieved with `decltype` however: lambda as data member.
Indeed, each lambda expression as a unique type so `decltype([]{}) foo = []{};` can't work,
and because of that lambda as data member cannot be achieved, unless of course by resorting to some
kind of type erasure, for example `std::function`.

I suppose there isn't much value in using lambdas instead of member functions.
Except that, lambdas having capture group, you could store variables specific to a single callable within the capture group,
giving you less data member to care about.

For example, the following example captures a global variable (again, don't try this at home!) at construction time.

{{< ce compiler="autonsdmi" >}}/*
  prints 10 9 8 7 6 5 4 3 2 1
*/
#include <vector>
#include <iostream>
#include <range/v3/view/reverse.hpp>

int counter = 0;
struct object {
    auto id = [counter = ++counter] { return counter;};
};

int main() {
    std::vector<object> v(10);
    for(auto & obj : v | ranges::view::reverse) {
        std::cout << obj.id() << ' ';
    }
}
{{< /ce >}}


## So... why are auto NSDMI not in the standard?

They apparently almost got in in 2008, there were some concerns so they were removed and a bit forgotten a bit, despite
[N2713](https://wg21.link/n2713) proposing to add them.

When parsing a class, the compiler first parses the declarations (functions signatures, variables definitions, nested classes, etc),
then parse the inline definitions, method default parameters, and data member initializers.

That lets you initialize a member with an expression depending on a member not yet declared.

{{< ce compiler="autonsdmi" >}}
struct s {
    int a = b();
    int b();
};
{{< /ce >}}

However, if you introduce auto members, things aren't that simple. Take the following valid code

{{< ce compiler="autonsdmi" >}}
struct s{
    auto a = b();
    int b() {
        return 42;
    };
} foo;
{{< /ce >}}

Here, what happens is

1. The compiler creates a member `a` of `auto` type, at this stage the variable `a` has a name, but no actual, usable type.

2. The compiler creates a function `b` of type int;
3. The compiler parses the initializer of `a` and `a` becomes an `int`, however, `b()` is not called.
4. The compiler parses the definition of `b`
5. The compiler construct foo and calls `b()` to initialize `a`

In some cases, the class is not yet complete when the compiler deduces a data member type, leading to an ill-formed program:

{{< ce compiler="autonsdmi" >}}
struct s {
    auto a = sizeof(s);
    auto b = 0;
};
{{< /ce >}}

Here:

1. The compiler creates a member `a` of `auto` type, at this stage the variable `a` has a name, but no actual, usable type.
2. The compiler  creates a member `b` of `auto` type
3. The compiler parses the initializer of `a` in order to determine its type
4. At this stage, neither the size of a or b is known, the class is "incomplete" and `sizeof` expression is ill-formed: `error: invalid application of 'sizeof' to an incomplete type 's'`.

So there are certain things you can not do within auto-nsdmi: calling `sizeof` referring to `*this` (even in decltype), constructing an instance of the class, etc.
All of this makes sense and you would run with the same issue with `decltype`. Or simply by doing

{{< ce compiler="autonsdmi" >}}
struct s {
    s nope;
};
{{< /ce >}}


Another gotcha is that an `auto` data member cannot depend on another data member declared after:

{{< ce compiler="autonsdmi" >}}
struct s {
    auto a = b;
    auto b = 0;
};
int main() {
    return s{}.a;
}
{{< /ce >}}

Here:

1. The compiler creates a member `a` of `auto` type, at this stage the variable `a` has a name, but no actual, usable type.
2. The compiler creates a member `b` of `auto` type, at this stage the variable `b` has a name, but no actual, usable type.
3. The compiler parses the initializer of `a` in order to determine its type. the type of `b` is unknown and therefore the program is ill-formed.

Which again, should feel natural to most c++ developers. Alas, these quirks were enough for the feature never to make in the working draft.

### Binary compatibility

Changing `struct S { auto x = 0; };` to `struct S { auto x = 0.0 ; };` breaks abi compatibility. While this may indeed be a bit confusing, functions with `auto`
return type have the same issue. In general exposing binary-stable interfaces in C++ is a complicated exercise that should be avoided.
This proposed feature does not significantly exacerbate the issue.
If for some reason you care about binary compatibility, avoid using `auto` in your exported interfaces. And maybe avoid using *data member initializers* altogether.


## Is a paper coming?

It's not something I plan to do, I just wanted to start a discussion again!
The [original paper](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2008/n2713.html) is too old to be still relevant.

The author noted at the time:

> Recently, it was pointed out on comp.lang.c++.moderated that one can get the same effect anyway, just with uglier code, using decltype.
Because of that, the author believes that the objection to auto has softened.

The wording of the standard changed significantly since then. Enough that it took me a while to find what exactly prevents auto NSDMI in today's standard, so let's look at some wording.

> [dcl.spec.auto](http://eel.is/c++draft/dcl.spec.auto) The type of a variable declared using auto or decltype(auto) is deduced from its initializer. This use is allowed in an initializing declaration ([dcl.init]) of a variable. auto or decltype(auto) shall appear as one of the decl-specifiers in the decl-specifier-seq and the decl-specifier-seq shall be followed by one or more declarators, each of which shall be followed by a non-empty initializer.

That first paragraph makes `auto foo = ...` valid, and was easy to find. However, it says nothing about excluding data members ( nor explicitly allowing static data members).

> [basic](http://eel.is/c++draft/basic#6) A variable is introduced by the declaration of a reference other than a non-static data member or of an object. The variable's name, if any, denotes the reference or object.

I was stuck for quite some time before I thought of checking up the normative definition of `variable`, which single-out non-static data members. Neat.

So, adding auto NSDMI to the standard would only requiere to add:

> [dcl.spec.auto](http://eel.is/c++draft/dcl.spec.auto) The type of a variable <span style="background-color:#e6ffed;"> or data-member</span> declared using auto or decltype(auto) is deduced from its initializer. This use is allowed in an initializing declaration ([dcl.init]) of a variable.

But the committee may also want to specify exactly the way auto-NSDMI and late class parsing interact, which is easy enough to explain in a blog post but much harder to write wording for.

## Acknowledgments

* Matt Godbolt and the compiler explorer team for helping my put this experimental branch on compiler explorer.
* Faisal Vali who authored the initial clang support.
* Alexandr Timofeev who motivated me to write this article.

## References
* [N2713 - Allow auto for non-static data members - 2008](https://wg21.link/n2713)
* [N2712 - Non static data member initializers](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2008/n2712)
* [C++ Working draft](http://eel.is/c++draft/)
