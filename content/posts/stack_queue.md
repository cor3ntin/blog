---
title: "Stranded with a C++ compiler and a bunch of queues"
date: 2016-10-31
---

A friend had a phone interview for a job in a company that I won’t name
- It’s Microsoft. One of the questions was about describing how he would
write a stack, only using standard queues.

I was confounded, because long before an algorithm could form in my
mind, I already decided that there was no solution that would actually
be useful in any real life scenario.

{{< ce >}}
template <typename T, typename Container = std::queue<T>>
class stack {
public:
    void push(const T &);
    void pop();
    T& top();
    std::size_t size() const;
    bool empty() const;

private:
    void transfer();
    Container a, b;
};
template <typename T, typename Container>
void stack<T, Container>::push(const T& t) {
    a.push(t);
}

template <typename T, typename Container>
void stack<T, Container>::pop() {
    transfer();
    a.pop();
    std::swap(a, b);
}

template <typename T, typename Container>
void stack<T, Container>::transfer() {
    while(a.size() > 1) {
        T t = a.front();
        a.pop();
        b.push(t);
    }
}
{{< /ce >}}

That the only solution I could find; To be honest, I was too lazy to
come up with the algorithm myself, but it’s really straight forward.

It has $\mathcal{O}( n )$ complexity, and… let’s just say it does not really scale.

But, it’s quite an interesting algorithm nonetheless. See, for a huge
company to ask this question to every candidate, I can only assume one
former employee found themselves stranded on an island, with a bunch of
queues. Their survival depended on having a stack, they failed to come
up with the proper solution and died.

It’s the only explanation that make sense to me; The other explanation
would be that large companies ask really stupid & meaningless interview
questions, and, well... that’s just silly.

Then, my friend told me the next question was about creating a queue
using stacks.

Sure, why not ?
{{< ce >}}
template <typename T, typename Container>
class queue {
public:
    void push(const T &);
    void pop();
    T& front();
    std::size_t size() const;
    bool empty() const;

private:
    void transfer();
    Container a, b;
};
template <typename T, typename Container>
void queue<T, Container>::push(const T& t) {
    a.push(t);
}

template <typename T, typename Container>
void queue<T, Container>::pop() {
    transfer();
    b.pop();
}

template <typename T, typename Container>
void queue<T, Container>::transfer() {
    if(b.empty()) {
        while(!a.empty()) {
            T t = a.top();
            a.pop();
            b.push(t);
        }
    }
}
{{< /ce >}}


My friend and I debated about the complexity of this algorithm. I
explained to him it was n². If our hero was stranded on an island, they
could not have standard stacks shipped their way by amazon, and would
have had to use what they had: a stack made of queues.

Of course, our unfortunate hero had a stock of standard queues to begin
with, but maybe hey could’t use them, for some reason. After all, he
didn’t invent them himself so it was better to rewrite them anyway.

```cpp
template <typename T> using MyQueue = queue<T, stack<T>>;
```


By that point, the poor cast away recognize a knife would have been more
useful than a standard container and they realized their death was
nothing but certain.

And, as the hunger and their impending doom lead to dementia, they
started to wonder… can we go deeper ?

After all, it is good practice to have good, solid foundations, and a
bit of judiciously placed redundancy never hurts.

```cpp
template <typename T>
using MyQueue = queue<T, stack<T, queue<T, stack<T, std::queue<T>>>>>
```

{{< figure src="https://cdn-images-1.medium.com/max/800/1*-lkFAZJdBQPlFLtCfU86OQ.png" title="A tree representation of a stack based queue" >}}

The structure has the property of being self-tested and grows
exponentially more robust at the rate of 2\^n which could prove very
useful for critical applications. We can however lament that 4 levels is
a bit arbitrary and limited.

Fortunately, I made the assumption that our hero, has with them a C++
compiler. That may be a depressing consideration when you haven’t drink
for 3 days, but, isn’t meta programming fantastic ?

After a bit a tinkering, cursing and recursing, it is possible to create
a queue of stacks - or a stack of queue - of arbitrary depth.


{{< ce >}}
namespace details {
    template <typename T, typename...Args>
    struct outer {
        using type = queue<T, Args...>;
    };


    template <typename T, typename...Args>
    struct outer<T, stack<Args...>> {
        using type = queue<T, stack<Args...>>;
    };

    template <typename T, typename...Args>
    struct outer<T, queue<Args...>> {
        using type = stack<T, queue<Args...>>;
    };

    template <unsigned N, typename T>
    struct stack_generator {
        using type  = typename outer<T, typename stack_generator<N-1, T>::type>::type;
    };
    template <unsigned N, typename T>
    struct queue_generator {
        using type  = typename outer<T, typename queue_generator<N-1, T>::type>::type;
    };

    template <typename T>
    struct stack_generator<0, T> {
        using type = queue<T>;
    };

    template <typename T>
    struct queue_generator<0, T> {
        using type = stack<T>;
    };

    constexpr int adjusted_size(int i) {
        return i % 2 == 0 ? i+1 : i;
    }
}
template <typename T, unsigned N>
using stack = typename details::stack_generator<details::adjusted_size(N), T>::type;

template <typename T, unsigned N>
using queue = typename details::stack_generator<details::adjusted_size(N), T>::type;


{{< /ce >}}

They are pretty cool and easy to use:

```cpp
stack<int, 13> stack;
queue<int, 13> stack;
```

On the system it was tested with, $N=13$ was sadly the maximum possible
value for which the program would not crash at runtime - The deepest
level consists of 8192 queues. The compiler was unable to compile a
program for $N > 47$. At that point the generated executable weighted
merely 240MB

I expect these issues to be resolved as the present solution - for which
a Microsoft employee probably gave their life - gains in popularity.
However, for $N > 200$, the author reckon than the invention of
hardware able to withstand the heat death of the universe is necessary.

You may be wondering if you should use those containers in your next
application ? Definitively ! Here are some suggestions.

 - An internet enabled toaster : A sufficiently big value of $N$ should
let you use the CPU as the sole heating element leading to a to a
slimmer and more streamlined design, as well as reducing
manufacturing costs.

- In an authentication layer, as the system has a natural protection
against brute force attacks. N should be at least inversely
proportional to the minimum entropy of your stupid password creation
rules. The presented solution is however not sufficient to prevent
Ddos

- Everywhere you wondered if you should use a vector but used a linked
list instead.
