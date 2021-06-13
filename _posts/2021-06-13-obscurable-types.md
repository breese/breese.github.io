---
layout: post
title: Obscurable Types
tags: C++
---

Imagine if we could devise classes whose interface can be turned on or off at specific source code locations.
This could be used to build interfaces that are only available inside a transactional scope.
Attempting to use the interface outside the transactional scope would result in a compiler error.

<!--more-->

## Prologue

In [Almost Affine]({% post_url 2021-06-06-almost-affine %}) we explored how to use compile-time counters to create
affine functions that can only be invoked once.

Assume the existence of a compile-time counter.

{% highlight c++ %}
template <typename T, typename InstanceTag>
struct constexpr_counter
{
    // Returns current counter value.
    static constexpr T current();

    // Increments counter and returns old counter value
    static constexpr T next();
};
{% endhighlight %}

An affine function can be created by constraining the call operator on the counter value.

{% highlight c++ %}
template <int Instance = global_counter::next()>
struct function_delta
{
    using invoke_counter = instance_counter<Instance>;

    // Only invocable once
    template <int C = invoke_counter::next()>
    void operator()() requires (C == 0)
    {
        // Do stuff
    }
};

void use()
{
    function_delta fn;
    fn(); // Same as fn.operator()<0>()
    fn(); // Same as fn.operator()<1>() - fails to compile
    fn(); // Same as fn.operator()<2>() - fails to compile
}
{% endhighlight %}

We are going to extend this technique to use the counter across several member functions.

## Staged Type

Consider the case of two-step initialization, where the constructor does the first part of the initialization and
an `init()` function completes the initialization.

We would like to get a compiler error if we use the other member functions before the initialization has been completed.
We would also like a compiler error if we attempt to invoke the `init()` function twice.

{% highlight c++ %}
template <int Instance = global_counter::next()>
struct staged_type
{
    using init_counter = instance_counter<Instance>;

    // Only invocable once
    template <int C = init_counter::next()>
    void init() requires (C == 0)
    {
        // Do initial stuff
    }

    // Only invocable after init() has been invoked
    template <int C = init_counter::current()>
    void call() requires (C == 1)
    {
        // Do normal stuff
    }
};
{% endhighlight %}

The trick is that the `init()` function increments the compile-time counter just like the affine function,
and the `call()` function is constrained by the current counter value.

{% highlight c++ %}
void use()
{
    staged_type obj;
    obj.init();
    obj.call();
    obj.call();
}

void use_no_init()
{
    staged_type obj;
    obj.call(); // Fails to compile as init() not called first
    obj.call(); // Also fails
}

void use_double_init()
{
    staged_type obj;
    obj.init();
    obj.init(); // Fails to compile
}
{% endhighlight %}

## Obscurable Type

The staged type is an example of an obscurable type, where parts of the interface is enabled or disabled depending
on the use.

Let us create a class with functions to enable and disable the interface.
Both functions increments a compile-time counter, and the interface is only enabled when the counter is odd.

{% highlight c++ %}
template <int Instance = global_counter<int>::next()>
class obscurable_type
{
    using obscurable_counter = instance_counter<int, Instance>;

public:
    // Turn interface on.
    template <int C = obscurable_counter::next()>
    void enable() requires (C % 2 == 0)
    {};

    // Turn interface off.
    template <int C = obscurable_counter::next()>
    void disable() requires (C % 2 == 1)
    {};

    // Obscurable function
    template <int C = obscurable_counter::current()>
    void call() requires (C % 2 == 1)
    {
        // Do stuff
    }
};
{% endhighlight %}

The set of enabled and disabled functions will alternate as the compile-time counter increases.

Counter | `enable()` | `disable()` | `call()`
0 | &#x2705; | &#x274c; | &#x274c;
1 | &#x274c; | &#x2705; | &#x2705;
2 | &#x2705; | &#x274c; | &#x274c;
3 | &#x274c; | &#x2705; | &#x2705;
... | | |

The constraint used for `call()` can also be applied to other member functions we may add.

The enable and disable functions have been constrained by the compile-time counter to disallow nested use.
If nesting is required, then two compile-time counters are needed. One counter keeps track of
`enable()` calls and the other counter tracks `disable()` calls.
The `call()` function is disabled when these two counters are equal.

{% highlight c++ %}
template <int EnableInstance = global_counter<int>::next(),
          int DisableInstance = global_counter<int>::next()>
class nested_obscurable_type
{
    using enable_counter = instance_counter<int, EnableInstance>;
    using disable_counter = instance_counter<int, DisableInstance>;

public:
    // Turn interface on.
    template <int E = enable_counter::next()>
    void enable()
    {};

    // Turn interface off.
    template <int E = enable_counter::current(),
              int D = disable_counter::next()>
    void disable() requires (E < D)
    {};

    // Obscurable function
    template <int E = enable_counter::current(),
              int D = disable_counter::current()>
    void call() requires (E < D)
    {
        // Do stuff
    }
};
{% endhighlight %}

The enable and disable functions performs no run-time operations, but they could...

## Transaction Lock

We can demonstrate the above-mentioned technique with a concrete example.
Let us create a lockable type that prevents us from invoking a function unless the mutex is locked.

The lock and associated unlock functions define a transactional scope.
The lockable type also has an `invoke()` function that only is invocable inside the transactional scope.

{% highlight c++ %}
#include <functional> // std::invoke

template <typename Mutex,
          int Instance = global_counter<int>::next()>
class transaction_lock
{
    using transaction_counter = instance_counter<int, Instance>;

public:
    transaction_lock(Mutex& m) : mutex(m) {}

    // Begins transactional scope.
    template <int C = transaction_counter::next()>
    void lock() requires (C % 2 == 0) { mutex.lock(); };

    // Ends transactional scope.
    template <int C = transaction_counter::next()>
    void unlock() requires (C % 2 == 1) { mutex.unlock(); };

    // Invokes a function inside transactional scope.
    template <typename F,
              typename... Args,
              int C = transaction_counter::current()>
    auto invoke(F&& f, Args&&... args) requires (C % 2 == 1)
    {
        return std::invoke(std::forward<F>(f), std::forward<Args>(args)...);
    }

private:
    Mutex& mutex;
};
{% endhighlight %}

{% highlight c++ %}
#include <mutex>

void use()
{
    std::mutex mutex;
    transaction_lock<std::mutex> guard(mutex);

    // guard.invoke([] { /* Do stuff */ }); // Fails to compile

    guard.lock();
    // guard.lock(); // Fails to compile

    guard.invoke([] { /* Do stuff */ });
    guard.invoke([] { /* Do stuff */ });

    guard.unlock();
    // guard.unlock(); // Fails to compile

    // guard.invoke([] { /* Do stuff */ }); // Fails to compile
}
{% endhighlight %}

The obscurable types do not check whether the transactional scope is open when the object is destroyed.
It would be useful if we could check the transaction counter in the destructor, but to obtain the counter
we would have to turn the destructor into a template function which is not possible.

## Caveat

The same severe caveats from [Almost Affine]({% post_url 2021-06-06-almost-affine %}) also applies to the above.
