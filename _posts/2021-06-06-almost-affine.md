---
layout: post
title: Almost Affine
tags: C++
---

Affine objects can be [used at most once](https://en.wikipedia.org/wiki/Substructural_type_system).

We are going to build an affine function object that can only be invoked once.

<!--more-->

## Prologue

Let us start with a simple function object. The call operator takes no parameters and return nothing for notational convenience, but the techniques described below also applies when the call operator takes parameter and returns a value.

{% highlight c++ %}
struct function_alpha
{
    // Call operator
    void operator()()
    {
        // Do stuff
    }
};

void use()
{
    function_alpha fn;
    fn();
    fn();
}
{% endhighlight %}

<span class="marginnote">
C++20 [constraints](https://en.cppreference.com/w/cpp/language/constraints) are used for notational convenience. The constraints can also be written with [`std::enable_if`](https://en.cppreference.com/w/cpp/types/enable_if).
</span>
We can constrain the call operator to only be available under certain conditions.
The constraint below is not really useful yet; it just serves to demonstrate how it can be done.

{% highlight c++ %}
struct function_beta
{
    template <int C = 0>
    void operator()() requires (C == 0)
    {
        // Do stuff if C == 0
    }
};

void use()
{
    function_beta fn;
    fn(); // Same as fn.operator()<0>()
    fn(); // As above
    fn.operator()<0>();
    fn.operator()<1>(); // Fails to compile because C == 1
}
{% endhighlight %}

We have managed to create a function object with a pretty useless constraint that can only be violated with an inconvenient call notation.
In order to make this constraint useful, we need to introduce compile-time counters.

## Compile-Time Counter

A compiler-time counter returns an increasing constexpr integer value every time it is used.

<span class="marginnote">
[CWG 2118: Stateful metaprogramming via friend injection](https://wg21.cmeerw.net/cwg/issue2118).
</span>
Compile-time counters use a trick to enable stateful meta-programming. While this trick is legal, it is also controversal.

We are going to evade the controversy and simply stipulate that compile-time counters are possible.
See the following articles for a detailed explanation:

* [How to Implement a Constant-Expression Counter in C++](https://b.atch.se/posts/constexpr-counter/).
* [How to Hack C++ with Templates and Friends](https://ledas.com/post/857-how-to-hack-c-with-templates-and-friends/).
* [Adding element to arrays and changing variables during compilation](https://lordsoftech.com/programming/adding-elements-to-arrays-and-changing-variables-during-compilation-imperative-meta-metaprogramming-in-c/).

We are going to assume a compile-time counter with two functions: one to increment the counter and another to read the current value.
Furthermore, an `InstanceTag` template argument is added to enable us to create multiple counters.

{% highlight c++ %}
template <typename T, typename InstanceTag>
struct constexpr_counter
{
    // Return current counter value.
    static constexpr T current();

    // Increment counter and return old counter value
    static constexpr T next();
};
{% endhighlight %}

A unique type for `InstanceTag` is needed to create a new counter. We could use the closure type of lambda expressions as they are guaranteed to be unique.

{% highlight c++ %}
using my_counter = constexpr_counter<int, decltype([]{})>;
{% endhighlight %}

<span class="marginnote">
[P0315: Wording for lambdas in unevaluated contexts](https://wg21.link/P0315)
</span>
However, the above is only legal as of C++20, so we are instead going to use a global compile-time counter to create unique types for our other counters.

{% highlight c++ %}
struct global_counter_tag;

using global_counter = constexpr_counter<int, global_counter_tag>;
{% endhighlight %}

The unique types are created by a template class that takes an integral template argument to capture the global counter.

{% highlight c++ %}
template <int> struct tag;

using tag_zero = tag<global_counter::next()>; // tag<0>
using tag_one = tag<global_counter::next()>; // tag<1>
{% endhighlight %}

Unique counters can now be created with this unique type.

{% highlight c++ %}
template <int Instance = global_counter::next()>
using instance_counter = constexpr_counter<int, tag<Instance>>;

// Examples

using first_counter = instance_counter; // instance_counter<0>
static_assert(first_counter::next() == 0);
static_assert(first_counter::next() == 1);

using second_counter = instance_counter; // instance_counter<1>
static_assert(second_counter::next() == 0);
static_assert(second_counter::next() == 1);
{% endhighlight %}

Notice how the `instance_counter` obtains a global count in the template argument list. This ensures that we get a new count when the template is instantiated.
If the `global_counter::next()` call had been moved into the template body then all instance counters would had gotten the same value and thus become aliases to the same counter.

## Affine Operator

We can now use the compile-time counter to make call operator constraint on our function object more useful.

If you are thinking the technique described below has issues then you are probably right. See the Caveat section towards the end.

Recall that multiple invocations of the function would result in the same templated call operator being used.
{% highlight c++ %}
void use()
{
    function_beta fn;
    fn(); // Same as fn.operator()<0>()
    fn(); // Same as fn.operator()<0>()
    fn(); // Same as fn.operator()<0>()
}
{% endhighlight %}

The compile-time counter can be used to automatically generate different templated call operators every time we invoke the function.
Notice that we are using two counters below. The global counter is used to create an invocation counter that is unique to a given instance of the function object.
This enables us to have multiple instances of the function, each with their own counter.

{% highlight c++ %}
template <int Instance = global_counter::next()>
struct function_gamma
{
    using invoke_counter = instance_counter<Instance>;

    template <int C = invoke_counter::next()>
    void operator()()
    {
        // Do stuff
    }
};

void use()
{
    function_gamma fn;
    fn(); // Same as fn.operator()<0>()
    fn(); // Same as fn.operator()<1>()
    fn(); // Same as fn.operator()<2>()
}
{% endhighlight %}

An affine function can be created by adding the constraint back in.

{% highlight c++ %}
template <int Instance = global_counter::next()>
struct function_delta
{
    using invoke_counter = instance_counter<Instance>;

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

## Affine Copy

What happens if we copy the affine function object?

{% highlight c++ %}
void use()
{
    function_delta fn; // decltype(fn) == function_delta<0>
    auto fn_copy = fn; // decltype(fn_copy) == function_delta<0>
    fn();
    fn_copy(); // Fails to compile
}
{% endhighlight %}

Invoking the call operator on the copy fails to compile because the two objects have the same type, and thus use the same instance counter.

Resetting the instance counter of the copy can only be done by constructing an object with a different instance template argument. In other words, copy construction should return an object of a different type. This cannot be done with a copy constructor.

We provide a copy function instead that returns a function object with an increased instance count and thus a new instance counter.

{% highlight c++ %}
template <int Instance = global_counter::next()>
struct function_epsilon
{
    using invoke_counter = instance_counter<Instance>;

    function_epsilon() = default;

    template <int C = invoke_counter::next()>
    void operator()() requires (C == 0)
    {
        // Do stuff
    }

private:
    // Only affine_copy is allowed to invoke copy constructor
    template <int Current, int Next>
    friend function_epsilon<Next> affine_copy(const function_epsilon<Current>&);

    // Disable normal copy construction
    function_epsilon(const function_epsilon&) = delete;

    template <int OtherInstance>
    function_epsilon(const function_epsilon<OtherInstance>& other)
    {
        // Copy stuff
    }
};

template <int Current, int Next>
function_epsilon<Next> affine_copy(const function_epsilon<Current>& self)
{
    return function_theta<Next>(self);
}

void use()
{
    function_epsilon fn; // delctype(fn) == function_epsilon<0>
    auto fn_copy = affine_copy(fn); // decltype(fn_copy) = function_epsilon<1>
    fn();
    fn_copy();
}
{% endhighlight %}

Copy assignment, move construction, and move assignment should be handled in a similar way.

## Caveat

We have used meta-programming tricks to ensure that a function object is invoked at most once.

The obvious problem is that an adversarial user can by-pass the compile-time counter by explicitly invoking the call operator with template parameter. We already did that in the `function_beta` usage example.

A more grave problem is that our meta-programming tricks are not aware of the run-time execution context so it cannot handle conditional statements, loops, or recursion correctly.

{% highlight c++ %}
void call(bool silent)
{
    function_gamma fn;
    if (silent)
    {
        try { fn(); } catch (...) {};
    }
    else fn(); // Fails to compile even though only one call is made
}
{% endhighlight %}

{% highlight c++ %}
void loop()
{
    function_gamma fn;
    while (true) { fn(); } // Compiles even though multiple calls are made
}
{% endhighlight %}

{% highlight c++ %}
void recurse(const function_gamma& fn, int level = 42)
{
    if (level == 0) return;
    recurse(level - 1);
    fn(); // Compiles even though multiple calls are made
}
{% endhighlight %}

Another problem is that this technique does not work well across translation units.
The compile-time counter works by counting template instantiations of some internal tag type.
The compiler cannot see the template instantiations in other translation units, so it will instantiate the same tag types in each translation unit.
In other words, each translation unit will increment the counter from zero.
