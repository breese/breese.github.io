---
layout: post
title: Lazy Evaluation - Operators
tags: C++, Template
---

Operator overloading provides a compact syntax for bind expressions.
Standard placeholders do not support member operators, but we can create our
own placeholders that works seamlessly with bind expressions.

<!--more-->

## Introduction

<p class="margintext">
Jaakko Järvi, Gary Powell, and Andrew Lumsdaine, “The Lambda Library: Unnamed Functions in C++”, Software: Practice and Experience 33(3), pp. 259-291, 2003.
<br>
Jaakko Järvi, <a href="https://www.boost.org/doc/libs/latest/doc/html/lambda.html">Boost.Lambda</a>, 1999-2004.
</p>
Overloaded operators for bind expressions predates the introduction of lambda expressions
in C++11 by a decade.

Suppose we want to sort a numeric container by their absolute value.
We can use `std::sort` with a comparison lambda expression.

{% highlight c++ %}
sort(first, last, [](auto a, auto b) { return a * a < b * b; });
{% endhighlight %}

Half of the above code is lambda boilerplate.
With overloaded operators the comparison bind expression can be written
with minimal boilerplating.

{% highlight c++ %}
sort(first, last, _1 * _1 < _2 * _2);
{% endhighlight %}

## Lazy Operators

The sorting example can be written using bind expressions directly.
{% highlight c++ %}
sort(first, last, bind(less{},
                       bind(multiplies{}, _1, _1),
		       bind(multiplies{}, _2, _2));
{% endhighlight %}

We can reduce this verbose syntax by using overloaded operators.

### Type Traits

We are going to use convenience type traits to check for a lazy type,
which is defined as either a bind expression or a placeholder, and to
check if there is at least one lazy type in a template parameter pack.

{% highlight c++ %}
template <typename T>
using is_lazy_expression =
  bool_constant<is_bind_expression_v<remove_cvref_t<T>> ||
                is_placeholder_v<remove_cvref_t<T>>>;

template <typename... Ts>
using any_lazy_expression = disjunction<is_lazy_expression<Ts>...>;
{% endhighlight %}

Assume we also defined the `_v` counterparts.

### Operator Overload

<p class="margintext">
Peter Dimov and Barry Revzin, "Adding functionality to placeholder types", <a href="https://wg21.link/P3171">P3171</a>, 2024.
</p>
P3171 proposes overloaded operators for bind expressions and placeholders.

{% highlight c++ %}
template <typename Lhs, typename Rhs>
constexpr auto operator+(Lhs&& lhs, Rhs&& rhs)
  requires any_lazy_expression_v<Lhs, Rhs>
{
  return std::bind(plus{}, forward<Lhs>(lhs), forward<Rhs>(rhs));
}
{% endhighlight %}

This means that we can create a bind expression by adding placeholders.
Assume `a` and `b` are normal variables with a numeric type such as `int`
or `float`.
{% highlight c++ %}
auto f = a + _1;

// becomes (via above overloaded operator+)
auto f = bind(plus{}, a, _1);
{% endhighlight %}

The resulting bind expression can then be invoked with another variable.
{% highlight c++ %}
auto r = f(b);

// becomes
auto r = plus{}(a, b);

// which becomes
auto r = a + b;
{% endhighlight %}

Operator overloading gives us a convenient way of creating bind expressions.
We have to overload the other operators as well, but leave that as an exercise for the reader.

### Compound Assignment Operator

The standard library does not contain function objects for compound assignment operators,
but they are proposed in P3171.
A function object for `operator+=` is used as an example.

{% highlight c++ %}
template <typename Lhs, typename Rhs>
constexpr auto operator+=(Lhs&& lhs, Rhs&& rhs)
  requires any_lazy_expression_v<Lhs, Rhs>
{
  return bind(via::plus_assign, forward<Lhs>(lhs), forward<Rhs>(rhs));
}
{% endhighlight %}

The `via::plus_assign` function object is proposed by P3171 as `plus_equal` but
has been renamed here to avoid confusion with equality.

{% highlight c++ %}
namespace via {

inline constexpr struct {
  template <typename Lhs, typename Rhs>
  constexpr auto operator()(Lhs&& lhs, Rhs&& rhs) const
      -> decltype(declval<Lhs>() += declval<Rhs>())
  {
      return forward<Lhs>(lhs) += forward<Rhs>(rhs);
  }
} plus_assign{};

} // namespace via
{% endhighlight %}

The call operator has a trailing return type that acts as a constraint to disable
illegal usage, such as assigning to a const object.

We put all function objects in the `via` namespace. The next article will extend
them with argument-dependent lookup for lazy functions.
Operators already handle argument-dependent lookup so no special handling
is needed for compound assignment.

### Operator Duality

Notice that we can either do eager evaluation as usual by adding normal
variables, or lazy evaluation by adding lazy arguments.
This is simply how operator overloading works.

Example of eager evaluation using normal C++ rules
{% highlight c++ %}
auto r = a + b;
{% endhighlight %}

Lazy evaluation using our overloaded operator

{% highlight c++ %}
auto f = a + _1;
{% endhighlight %}

{% highlight c++ %}
auto r = f(b);
// becomes
auto r = a + b;
{% endhighlight %}

|![AXPY](/assets/lazy/axpy.png){: style="width:40%;"}|
|`_1 * _2 + _3`|
{: class="marginimage"}

### Nested Bind Expressions

Compound expressions become nested bind expressions.

{% highlight c++ %}
auto axpy = _1 * _2 + _3;

// becomes
auto axpy = bind(multiplies{}, _1, _2) + _3;

// which becomes
auto axpy = bind(plus{}, bind(multiplies{}, _1, _2), _3);
{% endhighlight %}

These expressions follow the normal [operator precedence](https://en.cppreference.com/cpp/language/operator_precedence)
rules, so we can use parentheses to change precedence.

|![Precedence](/assets/lazy/precedence.png){: style="width:40%;"}|
|`_1 * (_2 + _3)`|
{: class="marginimage"}

{% highlight c++ %}
auto expr = _1 * (_2 + _3);

// becomes
auto expr = _1 * bind(plus{}, _2, _3);

// which becomes
auto expr = bind(multiplies{}, _1, bind(plus{}, _2, _3));
{% endhighlight %}

With this we can achieve the goal set out in the introduction.

{% highlight c++ %}
// Sort using absolute values.
std::sort(first, last, _1  * _1 < _2 * _2);
{% endhighlight %}

But we are not quite done yet.

## Extended Placeholders

Some operators have to be defined as member operators. As we cannot extend the
standard placeholders with member operators, we have to create our own.
First a placeholder type is needed.

{% highlight c++ %}
namespace lazy {

template <int N>
struct placeholder {
  // Put member operators here
};

} // namespace lazy
{% endhighlight %}

We can register this type so it can be used anywhere we would use standard
placeholders.
<p class="margintext">
<em>A program may specialize this template for a program-defined type <tt>T</tt> to
have the base characteristics of <tt>integral_constant<int, N></tt> with <tt>N</tt> > 0 to
indicate that <tt>T</tt> should be treated as a placeholder type.</em>
<br/>
-- C++ Standard N4950, section [func.bind.isplace]
</p>
This is done by specializing `std::is_placeholder`.

{% highlight c++ %}
// Registers extended placeholders
namespace std {

template <int N> 
struct is_placeholder<lazy::placeholder<N>>
  : integral_constant<int, N> {};

} // namespace std
{% endhighlight %}

Notice that this trait resolves to an `integral_constant` rather than a
`bool_constant` like other traits.
It can still be used in a boolean context because it defines N = 0 to mean
no placeholder. So the first placeholder is N = 1, which explains the
one-based indexing of placeholders.

We use [user-defined literals](https://en.cppreference.com/cpp/language/user_literal)
like `1_p` for our placeholders to obtain a compact notation that is not
confused with standard placeholders.

{% highlight c++ %}
namespace lazy::placeholders {

template <char... C>
constexpr auto operator ""_p() -> placeholder<to_int<C...>::value> {
  return {};
}

} // namespace lazy::placeholders
{% endhighlight %}

The `to_int` template converts a string into an integer at compile-time.
The implementation is not important to understand placeholders, but can be
found in the appendix.

We also place all the overloaded operators in the `lazy` namespace.

Now we can write
{% highlight c++ %}
auto f = a + 1_p;

// becomes
auto f = bind(plus{}, a, 1_p);
{% endhighlight %}

### Member Operator

There are several member operators that would be useful to overload, but we only
show the overloaded assignment operator that returns a bind expression that can assign.

{% highlight c++ %}
template <int N>
struct placeholder {
  template <typename T>
  constexpr auto operator=(T&& t) const {
    return std::bind(via::assign, placeholder<N>{}, forward<T>(t));
  }
};
{% endhighlight %}

where `via::assign` is defined in the same way as `via::plus_assign` above

{% highlight c++ %}
namespace via {

inline constexpr struct {
  template <typename Lhs, typename Rhs>
  constexpr auto operator()(Lhs&& lhs, Rhs&& rhs) const
      -> decltype(declval<Lhs>() = declval<Rhs>())
  {
      return forward<Lhs>(lhs) = forward<Rhs>(rhs);
  }
} assign{};

} // namespace via
{% endhighlight %}

|![Assignment](/assets/lazy/assign-zero.png){: style="width:35%;"}|
|`1_p = 0`|
{: class="marginimage"}
Now we can create bind expressions that assigns to placeholders
{% highlight c++ %}
auto zero = 1_p = 0;

// becomes
auto zero = bind(via::assign, 1_p, 0);
{% endhighlight %}

which can be used as

{% highlight c++ %}
zero(a);

// becomes
via::assign(a, 0);

// which becomes
a = 0;
{% endhighlight %}

All of the above is essentially an implementation of the P3171 proposal
outside the `std` namespace.
We go beyond this proposal in the next article on lazy functions.

## Appendix

A possible implementation of a compile-time string to integer conversion
that works for C++11 constexpr.

{% highlight c++ %}
constexpr int to_int_impl(int result) {
  return result;
}

constexpr int to_int_impl(int result, char c) {
  return 10 * result + (c - '0');
}

template <typename... Ts>
constexpr int to_int_impl(int result, char c, Ts... tail) {
  return to_int_impl(to_int_impl(result, c), tail...);
}

template <char... C>
struct to_int {
  static const int value = to_int_impl(0, C...);
};
{% endhighlight %}
