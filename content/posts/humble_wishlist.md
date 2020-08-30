---
title: "To humbly present a wish-list for C++23"
date: 2020-04-12T16:54:58+02:00
---

In Prague, the committee adopted
[To boldly suggest an overall plan for C++23](https://wg21.link/p0592r4),
a paper that lays a list of priorities WG21 should focus on for C++23.

The vote was almost unanimous. I voted against it.
I figured it would be interesting to explain why.

## The problem with plans

Plans have a nasty tendency to turn into deadlines and expectations.

There was an uproar when contracts did not ship (even though that was the right decision),
while coroutines have shipped with known issues and modules have shipped with little experience
with the feature as it is in the standard.
Of course, all of that was considered, and for example, there is some consensus
that the benefits of having coroutines now outweigh the cost of shipping them later.
These decisions usually don't age well, and they are difficult tradeoffs. Tradeoffs which are
mostly self-inflicted as C++ has a long release cycle (3 years), which paradoxically leaves little
time for individual proposals to mature.

Plans are not and should not be promises.
The C++ community should not expect what will be standardized by 2023.
Yet plans end up being promises.
Votes in the committee end up being influenced by the plan.
Plans are often less flexible than anyone hopes and may be blind to new information.

So on principle alone, due to the nature of C++ - which will almost certainly still be around in 50 years -
Any process which may involuntarily lead to haste makes my skin crawl.

## Let's look at P0592

### Modularisation of the standard library

It is hard to argue that the standard library should not be modular.
Modules would allow the committee to care less about file-level dependencies, implementers could get rid of `_Ugly`
names, etc.
And yet, the reality is much less exciting.
It would be an ABI break for implementers to put implementation details in modules. For some implementers,
it would further be an ABI break to have exported symbols in modules.
Most implementers are further committed to support multiple versions of C++ within the same standard library codebase,
which makes modularizing the standard library even more complicated.

The thing is standard library headers can already be imported (you can write `import <vector>;`)
and implementers can translate `#include <vector>` into `import <vector>;` - even if some won't as they are very keen about not breaking odr-violations ridden code.

With this set of constraints, the standard library can be modularized, in that we can make `import std.core;` do something,
but that would have a very limited benefit (nicer syntax) and some drawbacks (you have to remember
if `std::vector` is in `std.core` or `std.base`) - but in any case it would offer no technical benefits over `import <vector>;`.

Knowing that I do not think modularizing the standard library should be a priority.
We should wait for people to realize the cost of ABI and then work on a modularization that offers clear technical benefits.

### Executors

There is no doubt that [P0443 - A Unified Executors Proposal for C++](https://wg21.link/p0443),
is a very important piece of infrastructure that needs to be in the standard as soon as possible.

I wish it was called `Sender Receivers`. Executors (objects that can execute a function),
could be replaced by an `execute` customization point, which would simplify the design greatly.

`Sender Receivers` would be virtually unusable without a few other proposals, notably

* [Eliminating heap-allocations in sender/receiver with connect()/start() as basis operations](https://wg21.link/p2006)
* [Towards C++23 executors: A proposal for an initial set of algorithms](https://wg21.link/p1897r2)
* A customisation point mechanism such as [`tag_invoke`](https://wg21.link/p1895r0)
* An operator `co_await` for senders
* More work on cancellation

And probably a few other pieces. Notably, having a rich set of algorithms is very important.
Quite a lot of work!

### Coroutines support

Speaking of coroutines, it seems worthwhile to tie async coroutines with the work on sender-receivers.
Besides that, it would be nice to have `generator` in the standard - there is no proposal for that yet.
It would be a small proposal.

### Networking

As far as I can tell people have been working on what is now the networking TS since about [2005](https://wg21.link/n1925).
A lot of work and energy.
But C++ has changed a lot, the committee has changed a lot, and more importantly, operating systems
and requirements have changed.
And I quite strongly believe the networking TS is not ready.

#### ASIO

The networking TS is based on [ASIO](https://think-async.com/Asio), which is a great, well-maintained
open-source header-only library with a permissive license that you can use today with C++11 and up.

Unlike ASIO, the TS does not offer:

* SSL (or rather, TLS)
* Serial ports
* Signal handling

So what we are about to put into the standard is a stripped-down version of an existing library.

As such, the only benefits of the standard library are legal reasons and availability.
The standard is not meant to fix broken policies and legal departments.

It is hard to think that standardization would increase the portability or quality of implementation as
in all likelihood the people maintaining ASIO know networking better than standard implementers.

#### No TLS

The networking TS has no SSL/TLS support.
That should be a deal-breaker. [C++ Networking Must Be Secure By Default](https://wg21.link/p1860).
I think it would be quite irresponsible for any popular programming language to encourage developers to put their
users at risk.

#### Limited use cases out of the box

The network TS further has no support for higher-level protocols such as HTTP, QUIC, Websockets, and as such the use of third-party libraries will still be necessary.
This is by no mean a criticism of the network ts which has a well-defined scope, but I think it's important
to understand that "networking" may not include your favourite unicorn.

##### The networking TS is not a networking TS

Ultimately sockets are not very interesting or special.
They are files (ish).
What is interesting is that the operations in 99.9% of cases need to happen asynchronously and deal with many
files concurrently.
So a large chunk of the networking TS is spent defining foundational pieces of an asynchronous model and includes

* Executors
* Timers
* Strands
* Services

As well as:

* Buffers
* I/O functions

But all of that lives in `std::net` namespace and is not generalized at all.
These design choices come from the historical impossibility to perform file/io
and network/io in the same context efficiently on some platforms,
or the belief that async i/o had no benefit.

But that changed. And the standard should take note.
The C++ standard doesn't need 2 or 3 asynchronous models, it doesn't need a multitude
of contexts competing for hardware resources...

I wish C++ had 1 async model (senders/receivers) and one unified, general-purpose
async i/o framework for files, networking, timer, and other devices.

The networking TS is also very hard to use correctly, which mostly comes from its asynchronous model
which makes memory management, ownership and error managements harder to deal with than with sender receivers.

#### Cancellation and error management

Like `std::filesystem`, `std::net` has 2 overloads for everything, one that throws, one that does not (except when it does).
I dislike that type of interface.
Errors and results are passed through the same callbacks which are less efficient and necessitate the result to be default constructible.
The networking TS does also not support stop tokens, so cancellation is harder to deal with properly.

#### The networking TS is a C++14 library

The networking TS supports coroutines awkwardly as an afterthought, has many requirements but no concepts,
has forwarding headers instead of modules support, etc...

This poses the question of whether the standard library should be a unified library with a common design
or a collection of distinct libraries whose design reflect their origin and so far the networking TS is
of the latter category.

Given all of that, I do not think the networking TS is at this point a proposal that should be entertained
for C++23.

### Reflection

Reflection is certainly the most impactful large language proposal that the C++ committee can focus on.
Its design seems mostly done but there are enough details that seem to need resolving that it's anyone's
guess when it might ship.


## A personal wishlist for C++23

It's very hard to try to think about what should be standardized as opposed to what _I_ need or desire for myself.
any such list is bound to be accidentally self-centred. I'll give it a shot anyway.
I am avoiding mentioning my own papers as even if I think they are useful, it doesn't seem fair to try to judge their priority.


### Reconsider ABI stability.

I talked about [ABI before](/posts/abi/) and I will talk about it again. But it seems evident that whatever your opinion, more discussion is needed.
Notably, [Goals and priorities for C++](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2137r0.html)
is a very important paper.

### Better error management and wider freestanding support

Error management, in general, is hard. C++ shows that by being extremely inconsistent.
I would like to see more discussions on error management, whether that is `expected`,
`status_code`, deterministic exceptions, more efficient exceptions or something else,
it would be helpful to have some policy and consistency in this area.

Error management is often what prevents features to work in [freestanding environments](https://wg21.link/p0829r4).
Making more C++ work in more environments should be a goal!

### std::bad_alloc considered harmful

The best C++ features are those that keep true to the "don't pay for what you don't use" principle.
The existence of `std::bad_alloc` makes a lot of code less efficient, penalizing 99% of users - not necessarily because of the exceptions themselves but mainly because of exceptions safety of the standard library.
By making the standard default allocator non-throwing and making everything conditionally `noexcept`, a lot of performance could be gained.

### P1144 - Object relocation in terms of move plus destroy

Just like move semantic made a lot of code more efficient for free,
this proposal makes container operations a lot more efficient.
Probably the best bang-for-the-buck performance improvements the committee can work on!

### Sender-Receiver based asynchronous general-purpose I/O library

As I said, `Sender-Receiver` is a priority.
On top of that, we could start thinking about general purpose async I/O facilities with support for timers, files, processes and sockets in a concept-driven approach so that other device types can be easily plugged-in.
This would ensure the standard does [not accumulate several io contexts over time](/posts/executors/).
Many bits of the networking TS, including the buffer APIs, could be reused.
It is unlikely such an effort could bear fruits before 23.

### Better customization points

The Sender-Receiver proposal relies heavily on customization point objects and the
[`tag_invoke` mechanism](https://wg21.link/P1895), which is super clever but that I find really hard to use, and I can't help but think it needs to be a language feature.

Something like [Customization Point Functions](https://wg21.link/p1292r0), with the
ability to forward through multiple proxies would be great.

More generally, [taming ADL seems increasingly important](https://wg21.link/p0922r0).
On that line, I wouldn't be surprised if making the standard library operators hidden friends speeds up
compilation more than a shallow modularization.

### Reflection

Reflection is one of the rare features that cannot be emulated by library trickery and it
enables (and not just improves) many use cases.
It is a bit soon to know if it might land in 23, that might require a minor miracle but
it is worth focusing on.

### Some Unicode

Of course, someone in the Unicode study group would tell you that Unicode is important.
And while Unicode can be supported without language modification,
some issues with the core wording make it harder than it needs to be.
These can be improved.
Vendors buy-in to allow people to write UTF-8 applications is also ultimately necessary.

### Small proposals

Looking at previous C++ standards, small features like `[[nodiscard]]`, `make_unique`, `[[no_unique_address]]`
are often a driving force in new standards adoption and are more immediately impactful than big poster features.
C++20 was a huge release. Focusing on small proposals has a lot of value.
For example:

* More views (`enumerate`, `zip`, `product` are my favorites)
* Better parameter pack manipulation
* `constexpr` maths and c-string functions
* The pipeline operator

Of course, there are many great proposals to consider, including pattern matching (which as nice as it is isn't as fundamentally important as reflection), and domain-specific proposals (numeric, algebra, etc).