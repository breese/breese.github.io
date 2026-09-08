---
layout: post
title: Lazy Evaluation - Functions
tags: C++, Template
---

Lazy functions do not evaluate functions immediately, but return bind
expressions that evaluate the function bodies.
Lazy functions can be invoked with lazy arguments to form new bind
expressions.

<!--more-->

## Introduction

In the [previous article]({% post_url 2026-09-04-lazy-operator %}) we sorted a container by their absolute value using squared values.

{% highlight c++ %}
sort(first, last, 1_p * 1_p < 2_p * 2_p);
{% endhighlight %}

While that works, assuming no overflow occurs, it would be more natural to use the absolute function instead.

{% highlight c++ %}
sort(first, last, abs(1_p) < abs(2_p));
{% endhighlight %}

We are going to define lazy functions that take bind expressions and placeholders as arguments
and return bind expressions.
Lazy functions are a supplement to lazy operators.

## Lazy Function

Creating a lazy function is fairly simple.
There are two parts: a constrained overloaded function that returns a bind expression,
and a function object that forwards the call to the underlying functionality.
This is similar to the lazy operators where we had a constrained overloaded
operator and a function object, such as `lazy::operator+=` and `via::plus_assign`.

The absolute function is used as an example.
Other lazy functions are defined in the same way.
The constrained overloaded absolute function is defined as

{% highlight c++ %}
namespace lazy {

template <typename T>
  requires is_lazy_expression_v<T>
constexpr auto abs(T&& t) {
  return std::bind(via::abs, forward<T>(t));
}

} // namespace lazy
{% endhighlight %}

where the `via::abs` function object forwards the call to `std::abs`

{% highlight c++ %}
namespace via {

inline constexpr struct
{
  template <typename T>
  constexpr auto operator(const T& t) const {
    return std::abs(t);
  }
} abs{};

} // namespace via
{% endhighlight %}

Calling `lazy::abs` returns a bind expression that calls `std::abs`.

{% highlight c++ %}
auto f = lazy::abs(1_p);

// becomes
auto f = bind(via::abs, 1_p);
{% endhighlight %}

{% highlight c++ %}
auto r = f(a);

// becomes
auto r = via::abs(a);

// which becomes
auto r = std::abs(a);
{% endhighlight %}

The sorting example can now be written as

{% highlight c++ %}
sort(first, last, lazy::abs(1_p) < lazy::abs(2_p));
{% endhighlight %}

We are going to refine this to act more consistently with lazy operators.

### Function Duality

Recall that operators can be invoked eagerly or lazily depending on the
arguments provided.
This is a consequence of how operator overloading works.

We would like the same for functions, so that `via::abs` either does eager or
lazy evaluation.
The above implementation does not work for lazy evaluation because it calls
`std::abs` which only works for fundamental types and a few standard numeric
types like `std::complex<T>`.

Rather than calling the qualified `std::abs()` directly, we call the unqualified `abs()`
to find either `std::abs()` or `lazy::abs()`.

{% highlight c++ %}
namespace via {

inline constexpr struct
{
  template <typename T>
  constexpr auto operator(const T& t) const {
    using std::abs;
    return abs(t);
  }
} abs{};

} // namespace via
{% endhighlight %}

The unqualified `abs()` causes name lookup to search for overloaded functions in the
current and outer scopes.
It must not find `via::max` itself as that could cause infinite recursion,
but fortunately `via::max` has not been declared at the point of invocation and
is therefore not found.

The search also includes the namespaces of the arguments, which is called
[argument-dependent lookup](https://en.cppreference.com/cpp/language/adl)
and that is how it finds `lazy::abs`.

Argument-dependent lookup does not find `std::abs` for fundamental types because
these types do not reside in the `std` namespace.
This is solved by importing the `std::abs` symbol into the current scope
with the `using` statement.
The same trick is often used for `std::swap`.

The resulting `via::abs` is a
[customization point object](https://en.cppreference.com/cpp/standard_library/cpo).
Strictly speaking, we should also add an equality operator, but that is not necessary
for our purposes.

We can now use `via::abs` for eager evaluation

{% highlight c++ %}
auto r = via::abs(a);

// becomes (via using statement)
auto r = std::abs(a);
{% endhighlight %}

and for lazy evaluation

{% highlight c++ %}
auto f = via::abs(1_p);

// becomes (via argument-dependent lookup)
auto f = lazy::abs(1_p)

// which becomes
auto f = bind(via::abs, 1_p);
{% endhighlight %}

From now on, we will use `via::abs` for the absolute function.

{% highlight c++ %}
sort(first, last, via::abs(1_p) < via::abs(2_p));
{% endhighlight %}

### Associated Namespaces

Suppose we want to calculate the absolute difference.
First we create a bind expression to calculate the difference and then we call
the absolute function.

{% highlight c++ %}
auto absdiff = via::abs(1_p - 2_p);

// becomes
auto absdiff = via::abs(bind(minus{}, 1_p, 2_p));

// what happens next?
{% endhighlight %}

The argument to `via::abs` above is a bind expression so we want the name
lookup to find `lazy::abs`.
The type of a bind expression is unspecified by the standard, but we know
for sure that it does not reside in our `lazy` namespace.

<p class="margintext">
<em>For each argument type <tt>T</tt> in the function call [...]
if <tt>T</tt> is a class template specialization, its associated entities also
include: the entities associated with the types of the template arguments provided
for the template type parameters</em>
<br/>
-- C++ Standard N4950, section [basic.lookup.argdep]
</p>
Fortunately argument-dependent lookup searches the namespaces associated with the function
arguments, including template arguments.
The above bind expression is a template with lazy placeholders, which
causes argument-dependent lookup to search the `lazy` namespace.

{% highlight c++ %}
// continued from above

// which becomes (via argument-dependent lookup)
auto absdiff = lazy::abs(bind(minus{}, 1_p, 2_p));

// which becomes
auto absdiff = bind(via::abs, bind(minus{}, 1_p, 2_p));
{% endhighlight %}

This can be used as

{% highlight c++ %}
auto r = absdiff(a, b);

// becomes
auto r = bind(via::abs, bind(minus{}, 1_p, 2_p))(a, b);

// which becomes
auto r = via::abs(minus{}(a, b));

// which becomes
auto r = std::abs(a - b);
{% endhighlight %}

The above all hinges on the use of our lazy placeholder which is located in the
`lazy` namespace.
The above would not work with standard placeholders.

## Reseating

Calling a lazy bind expression with lazy arguments results in a new bind
expression where placeholders in the former have been substituted by the
lazy arguments. This is called reseating.

Assume that we have defined a lazy maximum function as described above.

{% highlight c++ %}
auto ramp = via::max(0, 1_p);

// becomes (via argument-dependent lookup)
auto ramp = lazy::max(0, 1_p);

// which becomes
auto ramp = bind(via::max, 0, 1_p);
{% endhighlight %}

Notice how the lazy evaluation of `via::max` results in a bind expression that
uses `via::max`.
This enables reseating.

|![Reseat ramp](/assets/lazy/reseat-diff.png){: style="width:100%;"}|
|Extending at placeholder|
{: class="marginimage"}

We can combine expressions into larger expressions.
For example, calling the above `ramp` expression with a bind expression as argument gives
us a reseated bind expression.

{% highlight c++ %}
auto ramp_diff = ramp(1_p - 2_p);

// becomes
auto ramp_diff = bind(via::max, 0, 1_p)(1_p - 2_p);

// which becomes (via reseating)
//   1_p is substituted with 1_p - 2_p
auto ramp_diff = bind(via::max, 0, 1_p - 2_p);
{% endhighlight %}

Reseating gives us the ability to construct new expressions by calling lazy bind
expressions with lazy arguments.
Extending lazy bind expressions via reseating only works at placeholder nodes
though.

### Eager Arguments

Invoking lazy bind expressions with eager arguments causes an eager evaluation
of the expression. This is what we ultimately want to use bind expressions
for.

|![Reseat](/assets/lazy/reseat-eager.png){: style="width:100%;"}|
|Contracting eager sub-expression|
{: class="marginimage"}

This also works for sub-expressions.
If a sub-expression only contains eager arguments, then that sub-expression is
evaluated eagerly and the result replaces the sub-expression.

{% highlight c++ %}
auto f = via::min(1_p, via::max(2_p, 3_p));

// becomes
auto f = bind(via::min, 1_p, bind(via::max, 2_p, 3_p));
{% endhighlight %}

{% highlight c++ %}
auto g = f(1_p, 22, 33);

// becomes
auto g = bind(via::min, 1_p, via::max(22, 33));

// which becomes (via eager evaluation)
auto g = bind(via::min, 1_p, 33);
{% endhighlight %}

This gives us the ability contract some parts of lazy bind expressions.
