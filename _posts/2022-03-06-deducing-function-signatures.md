---
layout: post
title: Deducing Function Signatures
tags: C++, Template
---

The signature of a function is a type consisting of the return type, the
function parameter types, and possibly function qualifiers.
The function signature is also known as the function type.

We demonstrate how to create a type traits that works reasonably well for
most callable objects.

<!--more-->

First we need to distinguish between a function type and a callable type.
A [callable type](https://en.cppreference.com/w/cpp/named_req/Callable) is
a type that can be invoked as a function. This includes function pointers,
lambda expressions, and function wrappers such as
[`std::function`](https://en.cppreference.com/w/cpp/utility/functional/function).
A function type is the type of the call operator that is invoked by the
callable type.

| Example | Function Pointer | Function Type |
|---:|:---:|:---|
| `int main()` | `int(*)()` | `int()` |
| `std::fminf(x, y)` | `float(*)(float, float)` | `float(float, float)` |
| `std::mutex::lock()` | `void (std::mutex::*)()` | `void()` |
| `[] {}` | `void (`*unspecified*`::*)() const` | `void() const` |

The function type does not include class information for member functions, even
though non-static member functions have an implied object argument, also known
as the `this` pointer.

## Motivation

Why should we care about the function type?

Suppose we have to implement our own variation of [`std::async()`](https://en.cppreference.com/w/cpp/thread/async)
that executes a task on a background thread. Our variation should use a pre-existing thread rather
than launching its own.
This pre-existing thread may be preoccupied with other tasks, so we enqueue our task for later execution.
We also want our variation to return an [`std::future`](https://en.cppreference.com/w/cpp/thread/future).

We must be able to store an arbitrary callable object in a queue, and this callable should return the
result in an `std::future`.
This is  precisely what [`std::packaged_task`](https://en.cppreference.com/w/cpp/thread/packaged_task)
is designed for.

This variation is really simple to implement.
<span class="marginnote">
We assume that the queue is thread-safe.
</span>

{% highlight c++ %}
template <typename Q, typename F, typename... Args>
auto async_push(Q& queue, F&& fn, Args&&... args) {

    // ?? is the function type of fn
    packaged_task<??> task(std::forward<F>(fn), forward<Args>(args)...);

    auto result = task.get_future();
    queue.push(std::move(task));
    return result;
}
{% endhighlight %}

The above does not compile because the `std::packaged_task` template requires a
function type, and it cannot be deduced from the callable type `F` using
[class template argument deduction](https://en.cppreference.com/w/cpp/language/class_template_argument_deduction),
nor do we have any standard transformation traits to deduce the function type from the
callable type.

The above challenge also applies when using other type-erased function wrappers such as
[`std::function`](https://en.cppreference.com/w/cpp/utility/functional/function)
or [`std::move_only_function`](https://en.cppreference.com/w/cpp/utility/functional/move_only_function).

## Function Type Trait

Our goal is to create a transformation trait that turns a callable type into a function type.

{% highlight c++ %}
template <typename T, typename... Args>
struct function_type;

template <typename T, typename... Args>
using function_type_t = typename function_type<T, Args...>::type;
{% endhighlight %}

The `Args...` argument types are needed for the template call operator and will
be explained later.

A callable can either be invoked directly or indirectly via its call operator.
The latter is going to complicate our transformation trait, so we start with
the easy case of directly invocable callables.
We introduce a helper template called `function_type_basis` to handle the
directly invocable callable types.

{% highlight c++ %}
template <typename>
struct function_type_basis {};

template <typename T, typename...>
struct function_type
    : function_type_basis<remove_cv_t<T>> {
};
{% endhighlight %}

<span class="marginnote">
We use C++20 concepts for brevity, but the function type trait can also be
implemented in C++11 using SFINAE.
In that case you should notice that that `noexcept` is not part of the function
type until C++17.
</span>
We are going to need a `function_constraint` concept that checks if a type is a
function type.
We can use the [`std::is_function`](https://en.cppreference.com/w/cpp/types/is_function)
type trait that does exactly this.

{% highlight c++ %}
template <typename T>
concept function_constraint = is_function_v<T>;
{% endhighlight %}

## Function Type

If the callable type is already a function type then we are done.

{% highlight c++ %}
template <function_constraint T>
struct function_type_basis<T> {
    using type = T;
};
{% endhighlight %}

## Function Pointers

If the callable type is a function pointer, then we remove the pointer to get
the function type.

{% highlight c++ %}
template <function_constraint T>
struct function_type_basis<T*> { // Notice the asterisk
    using type = T;
};
{% endhighlight %}

Similarly, if the callable type is a function reference, then we remove the
reference to get the function type. We have to handle both lvalue and rvalue
references.

{% highlight c++ %}
template <function_constraint T>
struct function_type_basis<T&> { // Notice the ampersand
    using type = T;
};

template <function_constraint T>
struct function_type_basis<T&&> { // Notice the ampersands
    using type = T;
};
{% endhighlight %}

The above matches the function [reference](https://en.cppreference.com/w/cpp/language/reference)
as in `R (&)(Args...)`, but not member function [reference qualification](https://en.cppreference.com/w/cpp/language/member_functions)
as in `R (C::*)(Args...) &`.
This is exactly what we want because we have to preserve all function qualifiers
to get the correct function type.

## Member Function Pointer

If the callable type is a member function pointer, then we remove the class
pointer to get the function type.

<span class="marginnote">
We use the `naive_` prefix to indicate that we are going to do something smarter
later on.
</span>
The naive implementation is to match the function type directly to deduce the
return type and the function parameter types.

{% highlight c++ %}
template <typename R, typename C, bool X, typename... Args>
struct naive_function_type_basis<R (C::*)(Args...) noexcept(X)> {
    using type = R(Args...) noexcept(X);
};
{% endhighlight %}

The above only matches member functions without function qualifiers, so we must
also match the other permutations of function qualifiers which leads to many
template specializations.

{% highlight c++ %}
// Matches lvalue-reference qualified member functions
template <typename R, typename C, bool X, typename... Args>
struct naive_function_type_basis<R (C::*)(Args...) & noexcept(X)> {
    using type = R(Args...) & noexcept(X);
};

// Matches rvalue-reference qualified member functions
template <typename R, typename C, bool X, typename... Args>
struct naive_function_type_basis<R (C::*)(Args...) && noexcept(X)> {
    using type = R(Args...) && noexcept(X);
};

// And so on for
//   const, const &, const &&,
//   volatile, volatile &, volatile &&,
//   const volatile, const volatile &, const volatile &&.
{% endhighlight %}

The above results in 12 template specializations, and that is not even exhaustive.
We need another 12 specializations to handle the permutations involving C-style
variadic arguments such as `R(Args..., ...)`.

Furthermore, if we have to support C++11/14 where `noexcept` is not part of the
function type, then we need to split each specialization into two; one with and
one without `noexcept`.
So we are potentially looking at 48 template specializations to deduce the
correct function type for member function pointers.

Fortunately we can use a more compact syntax instead of the naive member
function pointer matching above.
`T` becomes the function type when we match the member function type with
`T C::*`.

{% highlight c++ %}
template <function_constraint T, typename C>
struct function_type_basis<T C::*> {
    using type = T;
};
{% endhighlight %}

We are now able to handle all the directly invocable types with a total of
just 5 simple template specializations.

## Call Operator

Invoking function objects, including lambda expressions, results in a call to their
[call operator](https://en.cppreference.com/w/cpp/language/operators#Function_call_operator).
The function type of a function object is therefore the function type of its
call operator.

Consider a function object with a `void()` call operator.

{% highlight c++ %}
struct function_object_0001 {
    void operator()();
};
{% endhighlight %}

Deducing the function type is done in two steps

1. Deduce the callable type of the call operator. We select the call operator
   of class `C` by taking its address, that is `&C::operator()`, which gives
   us a member function pointer whose type can be found using the
   [`decltype` specifier](https://en.cppreference.com/w/cpp/language/decltype).
2. Deduce the function type from the callable type.
   The `function_type_basis` template from the previous section already does that.

The combined functionality can be implemented as:

{% highlight c++ %}
template <typename, typename...>
struct function_object_type {};

template <typename C, typename... Args>
    requires (bool(&C::operator()))
struct function_object_type<C, Args...>
    : function_type_basis<decltype(&C::operator())> {
};
{% endhighlight %}

The constraint checks if class `C` has a call operator.

We hook support for function objects into our general `function_type` template
with a template specialization.

{% highlight c++ %}
template <typename F, typename... Args>
    requires is_class<remove_cvref_t<F>>::value
struct function_type<F, Args...>
    : function_object_type<F, Args...> {
};
{% endhighlight %}

The template specialization is constrained to only accept a class type.
A [function object](https://en.cppreference.com/w/cpp/named_req/FunctionObject)
must be invocable, so this may have been a more obvious constraint to use,
but we would run into problems with nearly matching overloads if we try to
enforce that constraint here.
Instead we have made a weaker constraint by only requiring a class type.

The above implementation gives us the correct function type.

| Example | Function Type |
|:---|:---|
| `function_type_t<function_object_0001>` | `void()` |

## Template Call Operator

Our function type trait still does not support function objects with a template
call operators or generic lambda expressions, which for the purposes of this
discussion is the same.

Consider the following function object that has a template call operator.

{% highlight c++ %}
struct accumulator {
    template <typename... Args>
    auto operator()(Args... args) {
        return (args + ...); // Fold expression
    }
};
{% endhighlight %}

We want to deduce the function type given a set of arguments.
This requires a template specialization that uses the address of the call
operator where the argument types are explicitly added.

{% highlight c++ %}
template <typename C, typename... Args>
    requires (bool(&C::template operator()<Args...>))
struct function_object_type<C, Args...>
    : function_type_basis<decltype(&C::template operator()<Args...>)> {
};
{% endhighlight %}

We have to pass the argument types to the function type trait.

| Example | Function Type |
|:---|:---|
| `function_type_t<accumulator, int>` | `int(int)` |
| `function_type_t<accumulator, int, int>` | `int(int, int)` |

## Motivation Revisited

Now we can finally implement our `std::async` variation from the Motivation
section.

{% highlight c++ %}
template <typename Q, typename F, typename... Args>
auto async_push(Q& queue, F&& fn, Args&&... args) {

    using signature = function_type_t<F, Args...>;
    packaged_task<signature> task(std::forward<F>(fn), forward<Args>(args)...);

    auto result = task.get_future();
    queue.push(std::move(task));
    return result;
}
{% endhighlight %}

Notice that `std::packaged_task` does not accept a callable with function
qualifiers, such as `R(Args...) const` or `R(Args...) noexcept`.
This excludes us from passing lambda expressions among others, as we have no
standard transformation traits to modify such function types, but that is a
different topic for another time.

## Limitation

The function type trait we have created does not work with overloaded call
operators.
We would have to disambiguate the `&C::operator()` expression somehow, which
involves emulation of the overload resolution rules and that is a very deep
rabbit hole.
