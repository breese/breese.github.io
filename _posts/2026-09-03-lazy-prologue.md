---
layout: post
title: Lazy Evaluation - Prologue
tags: C++, Template
---

Lazy evaluation is deferred execution.
Lazy evaluation takes a function and some arguments, and returns a function
object that is capable of executing the given function at a later stage.

<!--more-->

## Introduction

This first article in a series on lazy evaluation using bind expressions
covers the preliminaries.
Subsequent articles explores topic such as lazy operators, lazy functions,
expression templates, and differentiation of lazy expressions.

Bind expressions are created by passing a function to
[`std::bind`](https://en.cppreference.com/w/cpp/utility/functional/bind.html).
So we start by understanding how operators or overloaded functions can be
passed as arguments to other functions.

## Function as Argument

A function can be passed as argument to another function.
Besides `std::bind` there are plenty of examples in the standard library
that takes functions as arguments, such as
[`std::apply`](https://en.cppreference.com/w/cpp/utility/apply.html),
[`std::visit`](https://en.cppreference.com/w/cpp/utility/variant/visit2.html),
[`std::function`](https://en.cppreference.com/cpp/utility/functional/function),
or [`std::thread`](https://en.cppreference.com/w/cpp/thread/thread/thread.html).

We use `std::apply` to find the largest element of a pair as an example.
`std::apply` is called with a function and a tuple, and invokes the function with
the tuple elements as function arguments.
[`std::max`](https://en.cppreference.com/w/cpp/algorithm/max.html) is an
obvious choice for the function.

{% highlight c++ %}
result = std::apply(std::max, pair_object); // Fails: Unresolved overload
{% endhighlight %}

Passing the function template `std::max` as an argument fails due to overload resolution.
Passing a function as an argument is done by passing the address of the function.
Overloaded functions and function templates are really many functions that each
have their own unique address, and the compiler asks us to choose which one.

We can choose a specific overloaded function by selecting the
[address](https://en.cppreference.com/cpp/language/overloaded_address) ourselves.
This can be done by casting the overloaded function to the appropriate function signature.

{% highlight c++ %}
result = std::apply(static_cast<const int& (&)(const int&, const int&)>(std::max),
                    pair_object);
{% endhighlight %}

This uses a complicated syntax that only works for one type.

## Function Object

<p class="margintext">
<em>If the postfix-expression is the address of an overloaded set, overload
resolution is applied</em>
<br/>
-- C++ Standard N4950, section $[$over.match.call.general$]$
</p>
The underlying problem is that passing an overloaded function as argument
causes overload resolution to kick in prematurely. We would like to defer
overload resolution until `std::apply` invokes the function with
the two pair elements, at which time the overloaded function can be
deduced with with the actual function parameters.

As an aside, suppose that we want to add the two pair elements rather than
choosing the largest element, then we would like the use `operator+`.

{% highlight c++ %}
result = std::apply(operator+, pair_object); // Fails: No address
{% endhighlight %}

This does not work because we cannot take the address of a built-in operator,
and even if we could, we would have the same problem with overloading as
before. The standard has long acknowledged this problem and provides a
function object wrapper for some operators. We can therefore use
[`std::plus`](https://en.cppreference.com/w/cpp/utility/functional/plus.html)
instead.
We ignore the minutiae of transparent function objects as they
are merely a historical accident.

{% highlight c++ %}
result = std::apply(std::plus{}, pair_object);
{% endhighlight %}

<p class="margintext">
<em>In places where one would expect to pass a pointer to a function to an
algorithmic template, the interface is specified to accept a function object.
This not only makes algorithmic templates work with pointers to functions,
but also enables them to work with arbitrary function objects.</em>
<br/>
-- C++ Standard N4950, section $[$function.object.general$]$
</p>
`std::plus` works because it is a [function object](https://cppreference.net/cpp/named_req/FunctionObject.html),
and `std::max` fails because it a function template.
Let us wrap `std::max` as a function object.
The function object is defined as an inline variable of a non-template class
with a template call operator.

{% highlight c++ %}
inline constexpr struct
{
  template <typename... Args>
  constexpr auto operator()(Args&&... args) const
  {
    return std::max(forward<Args>(args)...);
  }
} my_max{};
{% endhighlight %}

This function object can be used with `std::apply`

{% highlight c++ %}
result = std::apply(my_max, pair_object);

// becomes
result = my_max(pair_object.first, pair_object.second);

// which becomes
result = std::max(pair_object.first, pair_object.second);
{% endhighlight %}

The function object is an object, so the first argument for `std::apply` above
is an object, not the address of an overloaded function.
When the internals of `std::apply` invokes the object as a function, it uses
the `my_max` call operator with the pair elements, which calls `std::max` and
triggers overload resolution.
Using function objects therefore gives us a mechanism to defer overload
resolution until needed.

### Lambda Expression

We can also pass lambda expressions as function arguments, because lambda
expressions are essentially function objects.

{% highlight c++ %}
auto my_max = [] (auto lhs, auto rhs) { return std::max(lhs, rhs); };

result = std::apply(my_max, pair_object);

// becomes
result = my_max(pair_object.first, pair_object.second);

// which becomes
result = std::max(pair_object.first, pair_object.second);
{% endhighlight %}

## Binding

The standard library contains several facilities for creating function objects
that binds functions and arguments together.

[`std::bind_front`](https://en.cppreference.com/cpp/utility/functional/bind_front)
is used for partial application.
It returns a function object that binds an underlying function and some arguments
(`bs...`).
When the resulting function object is invoked with additional arguments (`as...`),
the underlying function is called with the bound arguments followed by the
additional arguments.

{% highlight c++ %}
auto g = bind_front(f, bs...);
{% endhighlight %}

{% highlight c++ %}
// Usage
auto result = g(as...);

// becomes
auto result = f(bs..., as...);
{% endhighlight %}

`std::bind_back` works in the same way, except the underlying function is called
with the additional arguments followed by the bound arguments.

{% highlight c++ %}
auto g = bind_back(f, bs...);
{% endhighlight %}

{% highlight c++ %}
// Usage
auto result = g(as...);

// becomes
auto result = f(as..., bs...);
{% endhighlight %}

Both of these functions return function objects, but not bind expressions.
Those come from `std::bind`.

## Bind Expression

Before you declare bind expressions an arcane construct that has been
replaced by lambda expressions, please indulge me for a while.
Bind expressions have an untapped potential that we shall return to later on.

[`std::bind`](https://en.cppreference.com/cpp/utility/functional/bind) returns a
bind expression, which is a function object that binds an underlying function
and some bound arguments (`bs...`).
Invoking the bind expression with additional arguments (`as...`) calls
the underlying function with the *bound* arguments, not the additional arguments.
The bound arguments may or may not be transformed first using the additional
arguments.
This is an important distinction from partial application in the previous section.

{% highlight c++ %}
auto g = bind(f, bs...);
{% endhighlight %}

{% highlight c++ %}
// Usage
auto result = g(as...);

// becomes
auto result = f(T_a(bs)...);
{% endhighlight %}

where `T_a(bs)...` denotes the transformation of the bound arguments (`bs...`)
using the transformation arguments (`as...`).
The transformation can be one of the following four.

### Bound Arguments

Normal arguments are bound by copy and are not transformed.

An example using normal arguments
{% highlight c++ %}
int value = 42;
auto g = bind(f, value);

auto result = g();
// becomes
result = f(42);
{% endhighlight %}

Bound arguments are equivalent to lambda capture by copy.
{% highlight c++ %}
int value = 42;
auto g = [value] () { return f(value); };

auto result = g();
// becomes
result = f(42);
{% endhighlight %}

### Bound References

Binding by reference is done using [`std::reference_wrapper`](https://en.cppreference.com/cpp/utility/functional/reference_wrapper).
References are not transformed.

{% highlight c++ %}
int value = 42;
auto g = bind(f, ref(value));

// value could be changed here

auto result = g();
// becomes
result = f(value);
{% endhighlight %}

Bound references are equivalent to lambda capture by reference.
{% highlight c++ %}
int value = 42;
auto g = [&value] () { return f(value); };

auto result = g();
// becomes
result = f(value);
{% endhighlight %}

### Placeholders

Bind placeholders are denoted `_N`, where `N` is a literal integer value.
Placeholder `_N` will be substituted by the Nth transformation argument.
Placeholders use one-based indexing, so placeholder `_1` refers to the first
transformation argument.

|![Bind Expression](/assets/lazy/bind-expression.png){: style="width:35%;"}|
|`bind(f, _2, _1)`|
{: class="marginimage"}

An example using placeholder substitution
{% highlight c++ %}
using namespace std::placeholders;

auto g = bind(f, _2, _1);

auto result = g(a, b);
// becomes
//   _1 is substituted with a
//   _2 is substituted with b
auto result = f(b, a);
{% endhighlight %}

The `using` statement brings the placeholder symbols into the current scope so
we can refer to them as `_1` rather than `std::placeholders::_1`.
All further examples silently assume that this has been done.

Placeholders are equivalent to lambda parameters.

{% highlight c++ %}
auto g = [] (auto arg1, auto arg2) { return f(arg2, arg1); }

auto result = g(a, b);
// becomes
auto result = f(b, a);
{% endhighlight %}

### Nested Bind Expressions

Nested bind expressions are injected into the enclosing bind expression.

|![Nested Bind Expression](/assets/lazy/nested-bind-expression.png){: style="width:40%;"}|
|`bind(g, bind(f, _2, _1), _3)`|
{: class="marginimage"}

An example using nested bind expressions
{% highlight c++ %}
// bind expression
auto g = bind(f, _2, _1);

// nested bind expression
auto h = bind(g, _3);
// becomes
auto h = bind(bind(f, _2, _1), _3);

auto result = h(a, b, c);
// becomes
auto result = f(b, a, c);
{% endhighlight %}

There is no direct analogy of nested bind expressions in lambda expressions.
Nested bind expressions gives us the ability to build expression trees.
