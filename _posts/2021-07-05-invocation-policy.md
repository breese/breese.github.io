---
layout: post
title: Invocation Policy
tags: C++
---

In C++ we value the [zero-overhead principle](https://en.cppreference.com/w/cpp/language/Zero-overhead_principle)
except when it comes to exceptions.
The standard C++ library generally prefers to raise exceptions when passed invalid arguments, and only in
a few select cases are such errors treated as undefined behavior to obtain better performance.
We are going to build a simple mechanism to let the user have a choice on our classes.

<!--more-->

## Wide or Narrow Contract

Functions must be invoked with valid parameters and on valid state to operate correctly.
Invalid parameters or state usually result in error handling, but for performance-critical functions
they may be considered precondition violations that result in undefined behavior.

Preconditions are constraints that must be satisfied when calling a function.
If the caller fails to meet these constraints, then no guarantees are given about how the
function may behave.
A function with precondition is said to have a narrow contract, and a function without has a wide contract.

<span class="marginnote">
[N3279: Conservative use of noexcept in the Library](https://wg21.links/N3279)
</span>
> *A wide contract for a function or operation does not specify any undefined behavior.
> Such a contract has no preconditions: A function with a wide contract places no
> additional runtime constraints on its arguments, or any object state, nor on any
> external global state.*

> *A narrow contract is a contract which is not wide. Narrow contracts for a functions or
> operations result in undefined behavior when called in a manner that violates the
> documented contract.*

Normally the library author decides whether a function has a wide or narrow contract.
In some cases they may offer both, such as [`std::array`](https://en.cppreference.com/w/cpp/container/array)
whose [`at()`](https://en.cppreference.com/w/cpp/container/array/at) function has a wide contract
and [`operator[]`](https://en.cppreference.com/w/cpp/container/array/operator_at) has a narrow contract.

Another example is [Boost.Outcome](https://www.boost.org/doc/libs/release/libs/outcome/doc/html/index.html)
that uses a consistent naming convention to distinguish between the two. For example,
[`value()`](https://www.boost.org/doc/libs/release/libs/outcome/doc/html/reference/types/basic_outcome/value_lvalue.html)
has a wide contract and
[`assume_value()`](https://www.boost.org/doc/libs/release/libs/outcome/doc/html/reference/types/basic_outcome/assume_value_lvalue.html)
has a narrow contract.

## An Example Container

We are going to demonstrate the invocation policy technique with a vector type.
Our vector type inherits from `std::vector` and overwrites the member functions that we want to change.

{% highlight c++ %}
#include <vector>

template <typename T>
struct my_vector : public std::vector<T>
{
private:
    using base = std::vector<T>;

public:
    using size_type = typename base::size_type;
    using const_reference = typename base::const_reference;

    using base::base; // Use base constructors

    // Overwrite member functions here
};
{% endhighlight %}

We are going to omit most of the above boiler-plate in the examples below for notational convenience,
as well as other important details like allocators, constexpr, and noexcept.
This should be considered for production code.

Let us warm up with a narrow contract variant of the `at()` function.

{% highlight c++ %}
template <typename T>
struct my_vector : public std::vector<T>
{
    // Expects: position < size()
    const_reference at(size_type position) const
    {
        // Narrow contract: requirements not checked.
        return base::operator[](position);
    }
};
{% endhighlight %}

We can change the function into a wide contract by checking the requirements.

{% highlight c++ %}
template <typename T>
struct my_vector : public std::vector<T>
{
    // Expects: position < size()
    const_reference at(size_type position) const
    {
        // Wide contract: requirements checked.
        if (position >= base::size())
	{
	    throw std::range_error("position exceeds capacity");
	}
        return base::operator[](position);
    }
};
{% endhighlight %}

Suppose we want to provide both the wide and narrow contract variants to the user.
We could adopt the before-mentioned Boost.Outcome naming convention.

{% highlight c++ %}
template <typename T>
struct my_vector : public std::vector<T>
{
    // Expects: position < size()
    const_reference assume_at(size_type position) const
    {
        // Narrow contract: requirements not checked.
        return base::operator[](position);
    }

    // Expects: position < size()
    const_reference at(size_type position) const
    {
        // Wide contract: requirements checked.
        if (position >= base::size())
	{
	    throw std::range_error("position exceeds capacity");
	}
        return assume_at(position);
    }
};
{% endhighlight %}

This enables the user to select the narrow contract variant for performance-critical code where
the requirements are known to be satisfied.

{% highlight c++ %}
template <typename T>
T summarize(const my_vector<T>& container)
{
    T sum = 0;
    for (std::size_t k = 0; k < container.size(); ++k)
    {
        sum += container.assume_at(k);
    }
    return sum;
}
{% endhighlight %}

Relying on a naming convention makes it difficult to distinguish between wide and
narrow constrants in meta-programming contexts though.
We would like to involve the C++ type-system somehow.

We are going to explore two alternative solutions.
1. Tag type that selects the proper overloaded function.
2. Template parameter that widens or narrows the function template as needed.

## Tag Types

Several standard C++ library facilities use tag types to select overloads.
This is especially common for constructors, where we cannot use a naming convension.

Tag types are empty types that have no member variables.
<span class="marginnote">
The technique also works with non-empty types, but we focus on empty types because they
can easily be optimized away.
</span>

{% highlight c++ %}
struct my_tag {};
{% endhighlight %}

Tag types are used at compile-time to select which code to generate.
There are plenty of examples of tag types in the C++ standard library.
Some of those are:

* [`std::nothrow_t`](https://en.cppreference.com/w/cpp/memory/new/nothrow_t) decides whether [`operator new`](https://en.cppreference.com/w/cpp/memory/new/operator_new) throws on out-of-memory or not.
* [`std::allocator_arg_t`](https://en.cppreference.com/w/cpp/memory/allocator_arg_t) disambiguates constructors that takes custom allocators.
* [`std::piecewise_construct_t`](https://en.cppreference.com/w/cpp/utility/piecewise_construct_t) and [`std::in_place_t`](https://en.cppreference.com/w/cpp/utility/in_place)
  disambiguates constructors that constructs member variables in-place.

Let us try to use tag types to distinguish between wide and narrow contracts.
<span class="marginnote">
Be advised though that there may be narrow contract functions that throws execeptions for other reason,
in which case it would be better to define your own tag type.
</span>
We are going to reuse the `std::nothrow_t` tag type because the examples are going to appear more natural.

We start by changing the `assume_at()` function to an overloaded `at()` function.

{% highlight c++ %}
template <typename T>
struct my_vector : public std::vector<T>
{
    // Expects: position < size()
    const_reference at(std::nothrow_t, size_type position) const
    {
        // Narrow contract: requirements not checked.
        return base::operator[](position);
    }

    // Expects: position < size()
    const_reference at(size_type position) const
    {
        // Wide contract: requirements checked.
        if (position >= base::size())
	{
	    throw std::range_error("position exceeds capacity");
	}
        return at(std::nothrow, position);
    }
};
{% endhighlight %}

Now the user can select the narrow contract variant by passing a `std::nothrow`.

{% highlight c++ %}
template <typename T>
T summarize(const my_vector<T>& container)
{
    T sum = 0;
    for (std::size_t k = 0; k < container.size(); ++k)
    {
        sum += container.at(std::nothrow, k);
    }
    return sum;
}
{% endhighlight %}

There are a couple of issues with the above.
1. We still cannot easily use the technique in a meta-programming context. We would have
   to add another overloaded function that takes a wide contract tag type, giving us a total
   of three functions per functionality.
2. We cannot apply this technique directly to `operator[]` because it takes exactly one
   argument so we cannot add the tag type argument.
   Instead of overloading on a tag type that is added to the argument list, we could
   introduce a wrapper type for the normal argument.
   If the normal argument is passed directly then the wide contract function is invoked,
   and if the normal argument is wrapped in our wrapper then the narrow contract function
   is invoked.

Resolving these issues adds complexity that we would like to avoid given that the library
author may want to use invocation policy for many functions on a class.
We are going to explore the template parameter solution instead.

## Function Template Parameter

In the tag type solution we added type information in the function argument list and
relied on overload resolution to select the invocation policy.
Another approach is to specify the invocation policy as a template parameter that
determines how the function behaves.

### Type trait

We start by defining a type-trait to check this template parameter.
We will use `void` to specify the default policy, which is the wide contract that checks
parameters and state. The narrow contract is selected with the `unchecked` type.

{% highlight c++ %}
#include <type_traits>

struct unchecked {};

template <typename>
struct is_checked_policy : std::true_type {};

template <>
struct is_checked_policy<unchecked> : std::false_type {};

template <typename T>
constexpr bool is_checked_policy_v = is_checked_policy<T>::value;
{% endhighlight %}

Notice that we can also use `std::nothrow_t` select the narrow contract as we did in the
previous section by adding a specialization of the type trait.

{% highlight c++ %}
#include <new> // std::nothrow_t

template <>
struct is_checked_policy<std::nothrow_t> : std::false_type {};
{% endhighlight %}

### Constexpr-if

We are going to use [constexpr if](https://en.cppreference.com/w/cpp/language/if) on the
type-trait above to determine whether or not to do run-time checking of invalid conditions.

{% highlight c++ %}
template <typename T>
struct my_vector : public std::vector<T>
{
    // Expects: index < N
    template <typename Policy = void>
    const_reference at(size_type pos) const
    {
        if constexpr (is_checked_policy_v<Policy>)
	{
            // Wide contract: requirements checked.
            if (pos >= base::size())
	    {
	        throw std::range_error("index exceeds capacity");
	    }
	}
        return base::operator[](pos);
    }
};
{% endhighlight %}

The function has a wide contract when invoked normally.

{% highlight c++ %}
template <typename T>
T summarize(const my_vector<T>& container)
{
    T sum = 0;
    for (std::size_t k = 0; k < container.size(); ++k)
    {
        // Invoke with wide contract
        sum += container.at(k);
    }
    return sum;
}
{% endhighlight %}

The function has a narrow contract when invoked by specifying the `unchecked` type
as template parameter.
As the member function is invoked with an explicit template parameter inside another template, we need to help
the compiler to [disambiguate the syntax](https://eli.thegreenplace.net/2012/02/06/dependent-name-lookup-for-c-templates)
by adding the `template` keyword to the invocation. This is not needed when
invoked outside a template.

{% highlight c++ %}
template <typename T>
T summarize(const my_vector<T>& container)
{
    T sum = 0;
    for (std::size_t k = 0; k < container.size(); ++k)
    {
        // Invoke with narrow contract
        sum += container.template at<unchecked>(k);
    }
    return sum;
}
{% endhighlight %}

The invocation can be quite a mouthful, but keep in mind that we only need to use
this when optimizating the code.

This technique also works for operators.

{% highlight c++ %}
template <typename T>
struct my_vector : public std::vector<T>
{
    // Expects: index < N
    template <typename Policy = void>
    const_reference operator[](size_type pos) const
    {
        if constexpr (is_checked_policy_v<Policy>)
	{
            // Wide contract: requirements checked.
            if (pos >= base::size())
	    {
	        throw std::range_error("index exceeds capacity");
	    }
	}
        return base::operator[](pos);
    }
};
{% endhighlight %}

The notation for specifying a template argument on an operator is even more inconvenient,
but it is doable.

{% highlight c++ %}
template <typename T>
T summarize(const my_vector<T>& container)
{
    T sum = 0;
    for (std::size_t k = 0; k < container.size(); ++k)
    {
        // Invoke with narrow contract
        sum += container.template operator[]<unchecked>(k);
    }
    return sum;
}
{% endhighlight %}

### Scoped Policy

We can define a type alias to apply the same invocation policy across multiple calls.

{% highlight c++ %}
template <typename T>
T diff_sum(const my_vector<T>& container)
{
    using scoped_policy = unchecked;

    T sum = 0;
    for (std::size_t k = 1; k < container.size(); ++k)
    {
        // Invoke with narrow contract
        sum += container.template at<scoped_policy>(k) -
	       container.template at<scoped_policy>(k - 1);
    }
    return sum;
}
{% endhighlight %}

