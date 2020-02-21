---
title: "A Universal I/O Abstraction for C++"
date: 2020-01-31T15:55:19+01:00
---

![Thread Spools](banner.jpg)

This article is the sequel to [A Universal Async Abstraction for C++]({{< ref "executors.md" >}}),
in which I talk about the [Executor proposal](https://wg21.link/P0443) targeting C++23.
Quite a bit happened since then.

SG-1[^1], the study group charged of all things concurrency and parallelism made forward progress
and sent the proposal to LEWG - with the hope of landing a future revision in the C++23 draft.
This is rather big news given that this work has been brewing for about a decade.

The split of `submit` into `connect` and `start` is now [the object of a paper](https://wg21.link/p2006). This is a very important piece of the puzzle and I look forward to seeing it discussed in Prague next month.

You can also read a short history of executors in [this paper](https://wg21.link/p2033r0.pdf).

Lastly, but maybe more importantly, Facebook published an open-source implementation of sender/receivers and scheduler called [libunifex](https://github.com/facebookexperimental/libunifex).
This is not an exact implementation of [P0443](https://wg21.link/P0443) and it has a lot more features and algorithms, but it implements the same basic design and architecture.
Unfortunately, it doesn't use concepts yet so I foolishly keep to trying to implement my C++20
library. And luckily Coroutines were merged into GCC and Concepts were merged into clang so there
are now many compilers that can implement the executors proposal.

Exciting times.

Last time we discussed two basic concepts:

* The `scheduler` concept which let you schedule an operation on a given context (such as a thread pool)
* The `executor` concept on which let you execute a function on a given context (such as a thread pool).
We saw how `executor` were rather not worthy of a concept as `executor::execute(Func)`
could simply be a <abbr title="Customization Point Object">CPO</abbr> for `submit(schedule(scheduler), std::as_receiver(Func))` [^2].

Being able to run code on an execution context, such as a thread is great.
But, what if you wanted to run code later? Maybe some piece for code needs to run every 5 minutes:

```cpp
void child() {
    while(true) {
        fmt::print("Are we there yet?");
        this_thread::sleep(5min);
    }
}
int main() {
    scheduler auto s = /*...*/
    execution::execute(s, as_receiver(child));
}
```

This will work[^14].
But nothing else will ever run on that thread which is a rather poor usage of resources.
Threads are less expensive than processes but they [still take time to create](https://lemire.me/blog/2020/01/30/cost-of-a-thread-in-c-under-linux/): Avoid having one thread per task if you have thousands of tasks.

What we would want is for the _task_ rather than the _thread_ to be interrupted for 5 minutes.

In fact, there are many instances when a task needs to wait, iddling a thread:

* Sleeping
* Waiting for data to be read from a socket or a file
* Waiting for a device to be flushed
* Waiting for a process to complete

All these operations can be are referred to as "I/O" and, on platforms with a kernel, they are usually handled by the kernel.

When calling the `::read` function, for example, the kernel will suspend the calling thread until
some data is available for that device and schedule another thread.
When data is available, the thread can be scheduled back.

This dance has a cost. A rather small one, you would need to create hundreds or thousands
of threads to notice.  Most of the cost probably comes from cache invalidation rather
than the context switch itself.

Instead of letting the kernel do the scheduling, there are system APIs that let us do the scheduling in user-space.

The basic principle is rather simple:

* Request the kernel to notify us when data is available on a file descriptor or handle
* Either
  * On another thread, wait for at least one request to complete
  * Check periodically that a request has completed.
* Run a callback associated to a request

## Asynchronous I/O APIs

### Reactors: select, poll, epoll

These POSIX (`epoll` is Linux specific) APIs have different behavior
which are not worth covering here as [Julia Evans covered that topic](https://jvns.ca/blog/2017/06/03/async-io-on-linux--select--poll--and-epoll/) better than I could.

Their principle is however identical:

* Register the file descriptor a task wishes to monitor
* Run some other task
* Call the API (ie call `select` on that set of files)
* It blocks until at least one file descriptor is ready to be read or written to
* Call the continuation (callback) associated with a file ready to be read
* Perform the necessary non blocking reads if enough data is available
* Repeat until all callbacks have been executed

This can happen either on a single thread (some tasks would be enqueued before the program starts waiting for file descriptors event) or happen across multiple threads, in which case we need to synchronize file registration. More on that later.

This general workflow is the **reactor** pattern.

### Proactors: AIO and IOCP

One problem with reactors is that for each `read` operation of a file, for example, we have to:

* Register the file (1 syscall)
* Poll until _some_ data is available (1 syscall)
* Repeat until enough data is available
* Read the data (in a non-blocking fashion) (1 syscall)

System calls are _relatively_ expensive, so is resuming tasks before they have enough data.
To palliate to that problem, more modern asynchronous I/O APIs such as `AIO` (POSIX) or IOCP (Windows), will merge the polling and reading operations.

This allows a more straightforward workflow:

* Register the file descriptor along with a set of buffers to fill
* Run some other task
* Suspend or periodically check that one or more I/O requests have completed
* Call the continuation (callback) associated with the completed request
* Repeat until all callbacks have been executed

This reduces the number of syscalls and lets us resume tasks only when the desired I/O have been fulfilled.
Internally the kernel may spawn its own pool of working threads to perform the I/O operations,
nothing is ever truly free.
However, this is a lot more efficient than performing more system calls.
This workflow is the **proactor** pattern.

But (There is always a but, isn't there ?).
While people have been doing Asynchronous I/O on Windows for ages (maybe because file operation on Windows are [painfully slow](https://github.com/microsoft/WSL/issues/873#issuecomment-425272829)),
`AIO` on Linux is either deemed unnecessary (synchronous I/O is fast enough) - or inadequate (too much latency).
In fact `AIO` on Linux is implemented in user-space - but a similar kernel APIs `io_submit` can be used instead. In any case, these APIs is designed to handle file i/o and it is either not possible or not recommended to use it for sockets as `epoll` would perform better in all cases.

Maybe more of interest to C++, people believe it was not possible to design an efficient interface that would cohesively handle both files and sockets.
Maybe this explains why we have both **ASIO** and **AFIO** as different projects with different interfaces, instead of some general asynchronous system, such as [`libuv`](https://libuv.org) or [Tokio](https://github.com/tokio-rs/tokio).

Beyoncé said that if you like it, you should put a ring on it[^3].
Well, I quite like senders/receivers and the idea of a standard general-purpose yet efficient
i/o scheduler, so maybe we should put a ring on it. More specifically, an `io_uring`.

### io_uring

`io_uring` is an exciting new feature in the Linux kernel which can allow the design of highly efficient, asynchronous frameworks that works just as well for (buffered and unbuffered) file I/O and other devices such as sockets.
`io_uring` was added to Linux 5.1[^4] as a replacement to `AIO` and `io_submit`,
but has since then improved support for sockets. It is so good it might morph into a general asynchronous system call interface.

`io_uring` is based on 2 queues (one for submission and one for completion) that are shared between the kernel.
The kernel can read from the submission queue while the application thread can read from the
completion queue even as the kernel writes to it.

<img src="uring.svg" alt="IO uring Proactor"/>

The queues are lock-free single consumer, single producer rings (hence the name).
Since Linux 5.5 the kernel will maintain an overflow list to hold completion until
there is space in the completion queue.

Similarly, the application must take care not to overflow the submission queue.
The submission queue can only be accessed by a single thread at once[^15].

Once work has been added to the ring, a single system `io_uring_enter` call can be used
to both submit all new work in the submission queue and wait for entries to be added to
the completion queue.

Here is a pseudo implementation of an i/o thread:

```cpp
void io_context::run() {
    io_uring ring;
    io_uring_queue_init(URING_ENTRIES, &ring, 0);
    struct io_uring_cqe* cqe;
    while(true) {
        add_pending_operations_to_io_uring();
        io_uring_wait_cqe(&ring, &cqe); // single syscall to submit and wait
        auto* operation = operation_from_completion(cqe);
        io_uring_cqe_seen(&ring, cqe);
        execute_completion(cqe);
    }
    io_uring_queue_exit(&m_ring);
}
```

This slide code features the [liburing](https://github.com/axboe/liburing) library
which handles the very low-level user-space ring management for us.


`run` can be executed on several threads, each with its own ring.
However, each queue can only be accessed from a single thread at once.
Moreover, `io_uring_wait_cqe` being, as the name suggests a blocking call,
how can we add work to the queue?

First, we need a thread-safe way to push an operation to the
submission queue buffer[^6] represented on the graphic above as a green rectangle.

```cpp
class io_context {
    std::mutex mutex;
    intrusive_queue<operation*> pending;
    void start_operation(operation* op) {
        std::unique_lock _(mutex);
        pending.push(op);
    }
};
```

But, if the i/o thread is currently blocked in an `io_uring_wait_cqe`,
how can it see that we added elements to the queue?

A naive solution is to use `io_uring_wait_cqe_timeout` but this has
a few issues:

* Entering and leaving the `io_uring` processing incurs a syscall and a context switch and more generally wastes CPU cycles.
* Depending on the value of the timeout, it would increase latency and causes a delay between
when the operation is started and when the kernel starts to execute the i/o request.

Instead, we can schedule a read operation on a dummy file handle in the io/thread,
and, in the sender thread, write to that file descriptor, which will cause the `io_uring_wait_cqe`
to return.

On Linux, we can use `eventfd`, which, as far as I can tell is the most efficient way to do that
little dance.

```cpp
class io_context {
    std::mutex mutex;
    std::queue<operation*> pending;
    int fd = ::eventfd(0, O_NONBLOCK);
    eventfd_t dummy;
    void run() {
        schedule_notify();
        while(true) {
            // --
            io_uring_wait_cqe(&ring, &cqe);
            if(cqe->user_data == this) {
            schedule_notify(); // re-arm
            }
            //...
        }
    }
    void schedule_notify() {
        auto sqe = io_uring_get_sqe(&m_ring);
        io_uring_prep_poll_read(sqe, fd, &dummy, sizeof(dummy));
        io_uring_set_data(sqe, this);
    }
    void start_operation(operation* op) {
        std::unique_lock _(mutex);
        pending.push(op);
        eventfd_write(fd, 0); // causes io_uring_wait_cqe to return
    }
};
```

This mechanism to enqueue work is not specific to `io_uring` and would also be used
with `epoll`, `select`, `io_submit`, etc.

#### Polling

This way of notifying the queue and waiting for completion events incur some overhead
which starts to be visible after a few hundred thousands of IOPS.
While this may not appear to be a problem, with newer standards such as PCI4/PCI5, and corresponding drives and network hardware, i/o starts to be CPU bound with the kernel being a bottleneck.

To this effect, `io_uring` provides a polling mode, which allows very high throughput in some use
cases. [P2052](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2052r0.pdf) advocate for
supporting such mode in the standard.

## The simplest I/O Operation: schedule_at

In [A Universal Async Abstraction for C++]({{< ref "executors.md" >}}), we discussed
the `schedule` algorithm which runs an operation on the execution context associated
with a given scheduler

```cpp
oneway_task do_something(execution::scheduler auto s) {
    co_await execution::schedule(s);
    fmt::print("Hello"); //runs in the context associated to the scheduler s
}
```

Now that we understand io contexts, aka execution contexts in which we can run
io operations, we can add a `deadline` parameter to the `schedule` [^7] algorithm.
I stole the idea of deadline from [P1031 - Low level file i/o library](https://wg21.link/p1031).
It is a simple utility which can represent a time, either relative or absolute

```cpp
task annoying_child(execution::scheduler auto s) {
    while(true) {
        //Suspend the task for 5 minutes,
        //The thread is free to do something else in the meantime
        co_await execution::schedule(s, 5min);
        fmt::print("Are we there yet?");
    }
}
```

Here, `execution::schedule(s, 5min);` returns a sender, like we saw last time for the `schedule`
algorithm.
The only difference is that the `start` method will lead to a timeout "i/o" operation being scheduled by the kernel.

<img src="schedule_deadline.svg" alt="IO uring Proactor"/>

`io_uring` happens to have built-in timeout support. Other scheduler may use [`timerfd`](http://man7.org/linux/man-pages/man2/timerfd_create.2.html) or [`CreateThreadpoolTimer`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-createthreadpooltimer?) on windows.

Beside timers, most Asynchronous APIS support:

* Reading, Writing to/from file descriptors (files, sockets, pipes, other "file-like" objects) in various modes
* Polling from file descriptors (waiting for data without actually reading it)
* Opening, syncing and closing file descriptors
* Connecting to a remote socket and accepting connections

While it is possible to imagine low-level APIs such as

```cpp
auto read_file(scheduler, native_handle, buffers) -> read_sender;
auto close_file(scheduler, native_handle) -> close_sender;
```

It is more likely that instead, we get few io objects such as `file`s and `socket`s

```cpp
template<execution::scheduler scheduler = std::default_scheduler>
class file;

task read_data(execution::scheduler auto s, buffers & buffs) {
    file f(s);
    co_await f.open("myfile.txt");
    co_await f.read(buffs);
    co_await f.close();
}
```

If you wonder why `f.close()` is not simply handled by RAII, read [P1662](https://wg21.link/p1662r0)
and weep.

## Threads are shared resources

There are a limited, fixed number of hardware threads, and unlike RAM,
it is not possible to [download more](https://downloadmoreram.com).

So ideally a program should use at most about the same number of frequently
active threads as there are active threads.

Unfortunately, independent libraries may use their own threads and thread pools.
I/O libraries might create their own even loops, as does pretty much every graphics framework.

The standard library use threads internally for parallel algorithms and `std::async`.
Under some implementations, there is a thread started for every `std::async` call (one of the many reasons why `std::async` is terrible).

And while we can transform 1000 elements of a vector one time,
it is harder to transform 1000 elements of 1000 vectors 1000 times at the same time. [Or something](https://en.wikipedia.org/wiki/La_Cité_de_la_peur).

This is why [P2079 - Shared execution engine for executors](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2079r0.pdf) makes the case for a globally accessible _execution_ context.

I do like that paper, but what we really need is a globally accessible _io context_. Or more specifically, a globally accessible _io scheduler_.

I/O contexts being a strict superset of execution contexts.

Before you make this face 😵 (which [might not be the right face](http://www.unicode.org/L2/L2019/19303-face-with-x-eyes.pdf)), confusingly horrified at the idea of adding a singleton to the standard, it is worth noting that some platforms reached the same conclusion long ago and expose a
global i/o context to all applications:

* Windows [Threads Pools](https://docs.microsoft.com/en-us/windows/win32/procthread/thread-pools) expose a default thread pool to which work - including io requests - can be submitted. This is used
by Microsoft's STL implementation.
* Apple platforms have [Grand Central Dispatch](https://en.wikipedia.org/wiki/Grand_Central_Dispatch), which works similarily but has a far cooler name.

There is no equivalent, de-facto solution on other POSIX platforms.
And while a one-thread context is simple enough, user-space scheduling is scheduling still, and
[scheduling is hard](https://tokio.rs/blog/2019-10-scheduler/).

There have some libraries that can be used on Linux such as `libdispatch` or `libuv`, or implementers can cook up something for scratch.

### Cancellation and stop tokens

Error management in C++ is considered a simple and solved problem[^8].
To spice things up, asynchrony adds a third channel: Cancellation.
Indeed, [cancellation is not an error](https://wg21.link/p1677r2)[^9].

But before we can talk about handling cancellation let's talk about emitting a cancellation
request.
You would typically cancel an entire task, or an operation, which would then cancel the
the entire chain of subsequent operations.

```cpp
sequence(read(stdin, buffer), write(stdout, buffer))
```

For example, here if we cancel the read, the write should not be executed.
As mentioned in [P1677] cancellation is the asynchronous version
of returning early from a function.

`std::stop_token` which is a C++20 feature that was accepted at the same
time as `std::jthread`[^10]

Like death and all good stories, asynchronous cancellation comes in threes:

* `stop_source`
* `stop_token`
* `stop_callback`

This is based on the same idea as [C#'s CancellationToken](https://docs.microsoft.com/en-us/dotnet/standard/threading/cancellation-in-managed-threads) and [Javascript's AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController).

`stop_source` can create tokens, `stop_token` has a `stop_requested` method that returns
true once `stop_source::request_stop()` is called.
In addition, callbacks can be triggered automatically when `stop_source::request_stop()`
is called.

<img src="stop.svg" alt="Stop Callback"/>

All tokens and callbacks attached to the same `stop_source` share the same
thread-safe ref-counted shared state.
(You are still responsible for making sure the functions used as `stop_callback` are themselves
thread-safe if you have multiple threads.)

It has already been implemented in GCC so you can play with it on compiler explorer

{{< ce_fragment compiler="gcc-trunk" >}}
{{< ce_code >}}
#include <stop_token>
#include <cstdio>

int main() {
    std::stop_source stop;
    auto token = stop.get_token();
    std::stop_callback cb(token, [] {
        std::puts("I don't want to stop at all\n");
    });
    std::puts("Don't stop me now, I'm having such a good time\n");
    stop.request_stop();
    if(token.stop_requested()) {
        std::puts("Alright\n");
    }
}
{{< /ce_code >}}
{{< /ce_fragment >}}

Tokens can then be attached to a coroutine task of the appropriate type [^11]
or attached to any receiver.

The customization point `execution::get_stop_token(execution::receiver auto)`
can then be used by an execution context to query whether to cancel the operation.

Operations should be canceled in the execution context on
which they are intended to be executed.

In the case of in-flight I/O operations, a request can be emitted to the kernel to
cancel the request (`CancelIo` on windows, `IORING_OP_ASYNC_CANCEL`, `aio_cancel`, etc).
Especially important to cancel timers, socket read or other operation that may never complete
otherwise.

### Execution contexts lifetime

At some point, I used a stop token to stop an execution context and cancel all the
tasks in flight. Which was super convenient.

That is, unfortunately, a recipe for disaster as canceling a task may cause it to be rescheduled
or another task to be scheduled on an execution context that might have been destroyed.
I have to admit, convincing me of that took a bit of effort (Thanks Lewis!).

Instead, executions contexts should not be destroyed until all operations that may run or schedule
other operations on that context are done.

This can be achieved by the `std::async_wait` algorithm which I mentioned in my first
blog posts about executors.

### Receivers and Coroutines asymmetries

It's all not roses though: There are a few mismatches between sender/receivers and awaitables/continuations.

Receivers have 3 channels: set_value, set_error and set_done representing respectively
success, failure, and cancellation.

Coroutines have a return value (which is of a single type - whereas receivers support multiple value types [P1341](https://wg21.link/p1341r0)) and can rethrow exceptions[^12].

Mapping receiver can then be achieved in a couple of ways:

1. **Returning some kind of `variant<ValueType, ErrorType, cancelled_t>`**

    ```cpp
    task example() {
        inspect(auto res = co_await sender) {
            <cancelled_t>: {

            }
            res.success():{

            }
            res.failure(): {

            }
        };
    }
    ```

    The above example showcases [Pattern Matching](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p1371r2.pdf), although I am not sure that we can mix both types and expressions matchers.

    We cannot use types to discriminate success and failure as they may have the same type.

 <p/>

2. **Use exceptions to propagate both errors and cancellation**

```cpp
task example() {
    try {
        co_await sender;
    }
    catch(const std::error_status&) {/*...*/}
    catch(const std::cancelled_operation&) {/*...*/}
}
```

This suffers a couple of issues:

* Semantic - Using exceptions to signal cancellation makes it look like cancellation is an error, which it is not. Such a zemblanity!

* Performance - The reliance on exceptions makes it even harder to use in embedded platforms as if the required heap allocations were not bad enough! Besides performance, sometimes the very support for exceptions is lacking.

But in truth, coroutines do not have to use exception to report different results.
This is a simplified diagram of a coroutine.
The coroutines is suspended and then resumed at a specific point represented by a continuation
handle.

<img src="coro1.svg" alt="Stop Callback"/>

We could imagine a coroutine having several possible continuations to resume at
depending on the result of the operation.

<img src="coro2.svg" alt="Stop Callback"/>

This would be a better modelization of receivers and would not suffer the performance and implementabilities issues of exceptions (at the cost of having more `coroutine_handle` to keep track of.)

Anyway... this has turned into blog-post driven design...

[Let's talk about a language that doesn't suck](https://www.destroyallsoftware.com/talks/wat), let's talk about Go.

## Gorroutines[^5] are not Goroutines

Goroutines, a feature of the Go programming language, are very different from C++ coroutines in that they are not only stackfull, but also model both a resumption mechanism and a scheduling mechanism.
Go provides you with a baked-in i/o and coroutines scheduler which will handle on behalf of the
program interrupting a goroutine when it performs an i/o, try to acquire a lock or any other blocking operation.

C++ coroutines are not Goroutines. **C++ Coroutines do not imply asynchrony, let alone scheduling**.
C++ is not the kind of language that will bake-in an i/o scheduler as it would go
against the "Don't pay for what you don't use" mantra and would make C++ unusable in many environments.

That being said...

The combination of coroutines, sender receivers, and i/o schedulers, can emulate goroutine (well, stacklessness non-withstanding).
C++ coroutines can also be used as simple synchronous generators. It is a much more general
and extensible system.

I think the end goal would be for every single potentially blocking call to be an
asynchronous expression instead. Like in `go`.
Not bake-in in the language mind you, but as library solutions.

For example, `libunifex` implement [async mutexes](https://github.com/facebookexperimental/libunifex/blob/master/include/unifex/async_mutex.hpp) (not unlike `asio`'s strands),
such that you can acquire a lock by resuming a coroutine:

```cpp
task s::f() {
    co_await m_mutex.lock();
    // Do stuff
    m_mutex.unlock();
}
```

### Channel your inner Gopher

Along Goroutines, go offers channels, which are one of the best features of Go.
Channels are, conceptually, relatively simple.
A channel is a multi-producers, multi-consummers queue.
Reading from the queue suspend the goroutine until data is available.
Writing can be either buffered (The written data is saved and the writer can continue on its
merry way) - or Unbuffered (The writer is suspended until a reader is ready to take the data).
Well...

```cpp
using namespace cor3ntin::corio;
template <execution::scheduler scheduler>
oneway_task go_write(scheduler sch, auto w) {
    int i = 10;
    while(i) {
        co_await sch.schedule(std::chrono::milliseconds(100));
        co_await w.write(--i);
    }
}

template <execution::scheduler scheduler>
oneway_task go_read(scheduler sch, auto r, stop_source& stop) {
    while(true) {
        int value = co_await r.read();
        std::cout << "Got value " << value << "\n";
        if(value == 0) {
            stop.request_stop();
            break;
        }
    }
}

int main() {
    stop_source stop;
    io_uring_context ctx;
    std::thread t([&ctx, &stop] { ctx.run(stop.get_token()); });

    auto c = make_channel<int>(ctx.scheduler());

    go_write(ctx.scheduler(), c.write());
    go_read(ctx.scheduler(), c.read(), stop);
    t.join();
}
```

Nothing C++ can't do!

My implementation of channels is not quite ready yet, and this article is already long enough.
I might come back to the implementation of channels, and the few utilities required to implement them, including `async_mutex`, the `on` algorithm and the `get_scheduler` customisation point!

## A great opportunity awaits

The year is 2020 and even consummer CPUs feature double digits number of cores,
storage offers 10GB/s read speeds and networks have to accommodate ever-growing traffic.

Faced with these challenges, some have contemplated user-space networking
or contend with costly to maintain spaghetti codebases.

For a long time, the C++ committee seemed to think that either async file I/O
didn't make sense or was fundamentally irreconcilable with networking.
This belief would lead to two inter-incompatible APIs in the standard, which would be
a nightmare in term of usability (aka ASIO and AFIO).

I don't care about performance as much as I care about the usability of interfaces.
For better or worse, faced with a choice between performance and ergonomy, the committee tends to prioritize performance[^13].

Fortunately, it seems that there is finally a way to resolve these divides:

* `iouring` offer very high performance I/O which doesn't discriminate on device type.
* Sender Receiver provides the composable, low-cost, non-allocating abstraction while offering a simple mental model for asynchronous operations lifetime.
* Coroutines make asynchronous i/o dead simple for the 99% use case.

Asynchronous Networking is nice.

Asynchronous I/O is better.

**AWAIT ALL THE THINGS!**

I'll leave you with a quote from [P2052 - Making modern C++ i/o a consistent API experience from bottom to top](https://wg21.link/p2052).

***Sender-Receiver is genius in my opinion. It’s so damn simple people can’t see just how game changing it is: it makes possible fully deterministic, ultra high performance, extensible, composable, asynchronous standard i/o. That’s huge. No other contemporary systems programming language would have that: not Rust, not Go, not even Erlang.*** ― Niall Douglas

Until next time, take care! Thanks for reading.

## Resources and References

**Kernel Recipes 2019: Jens Axboe - "Faster IO through io_uring”**
{{< youtube id="-5T4Cjw46ys" >}}

### Papers

[Efficient IO with io_uring](https://kernel.dk/io_uring.pdf), Jens Axboe

[P1897](https://wg21.link/p1897) - Towards C++23 executors: An initial set of algorithms - Lee Howes

[P1341](https://wg21.link/p1341) - Unifying Asynchronous APIs in the Standard Library - Lewis Baker

[P2006](https://wg21.link/P2006) - Eliminating heap-allocations in sender/receiver with connect()/start() as basis operations - Lewis Baker, Eric Niebler, Kirk Shoop, Lee Howes

[P1678](https://wg21.link/P1678) - Callbacks and Composition - Kirk Shoop

[P1677](https://wg21.link/P1677) - Cancellation is not an Error - by Kirk Shoop, Lisa Lippincott, Lewis Baker

[P2052](https://wg21.link/p2052) -  Making modern C++ i/o a consistent API experience from bottom to top - Niall Douglas

[P0443](https://wg21.link/p0443) -  A Unified Executors Proposal for C++ - Jared Hoberock, Michael Garland, Chris Kohlhoff, Chris Mysen, Carter Edwards, Gordon Brown, David Hollman, Lee Howes, Kirk Shoop, Eric Niebler

[P2024](https://wg21.link/p2024) - Bloomberg Analysis of Unified Executors - David Sankel, Frank Birbacher, Marina Efimova, Dietmar Kuhl, Vern Riedlin


[^1]: A group that is in fact not chaired by Jack O'Neill. I never went there by fear of speaking out of order. Legend says they eat at round tables and fight for the forks.
[^2]: A hill I'd rather not die on!
[^3]: Something you would learn in [Software Engineering at Google: Lessons Learned from Programming Over Time](https://www.amazon.com/Software-Engineering-Google-Lessons-Programming/dp/1492082791/), along with many great insights about software engineering.
[^4]: Linux 5.6 will come with many improvements such as a redesigned worker threads.
[^5]: Coroutines are sometimes called Gorroutines (with 2 Rs) after the name of the man who worked on them for the best part of a decade: Gor Nishanov. Thanks Gor!
[^6]: A name I did made up.
[^7]: I made up that too. libunifex uses `schedule_after(duration)` and `schedule_at(time_point)`
[^8]: It's not and never will be. [[P0709](https://wg21.link/p0709)] [[P1947](https://wg21.link/p1947)] [[P1886](https://wg21.link/P1886)] [[P1886](https://wg21.link/P1886)] [[P0824](https://wg21.link/p0824)] [[P1028](https://wg21.link/p1028)] [[P0323](https://wg21.link/p0323)]
[^9]: [P1677 - Cancellation is not an error](https://wg21.link/p1677r2) is a paper worth reading if only because it contains 54 instances of the word **serendipitous**.
[^10]: [`std::jthread`](https://en.cppreference.com/w/cpp/thread/jthread) is now the recommended way to start a thread in C++ - I think it would be fair to consider `std::thread` deprecated, and maybe reflect on how we got into this infortunate situation.
[^11]: [_Someone_](https://lewissbaker.github.io) should write a blog post about that...
[^12]: In fact, continuations in C++20 can never be `noexcept`, which is rather unfortunate.
[^13]: Try not to think about standard associative containers when reading that. Too late!
[^14]: If `main` doesn't return too soon which we can't prevent with `execution::execute` because [One-Way execute is a Poor Basis Operation](https://wg21.link/p1525r0)
[^15]: A first draft of this sentence read ***"The submission queue can only be accessed by a single thread concurrently"***. But `concurrent` is a too subtle word to be ever used properly by the mere mortal that I am.
