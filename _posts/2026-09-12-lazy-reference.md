---
layout: post
title: Lazy Evaluation - References
tags: C++, Template
---

Lazy references allows lazy bind expressions use normal variables.

<!--more-->

## Introduction

In a [previous article]({% post_url 2026-09-04-lazy-operator %}) we defined a lazy
placeholder to furnish it with a lazy assignment operator.
We would like to do the same for references.

Consider a bind expression that assigns to a normal variable `a`.
We must bind the variable by reference to modify it, which is done
with [`std::reference_wrapper`](https://en.cppreference.com/cpp/utility/functional/reference_wrapper)
in bind expressions.

{% highlight c++ %}
auto assign_to_a = bind(via::assign, ref(a), 1_p);
{% endhighlight %}

We would like a compact notation for this.

{% highlight c++ %}
auto assign_to_a = ref(a) = 1_p; // Fails: eager assignment
{% endhighlight %}

This fails because `std::reference_wrapper` does not have a lazy assignment operator.
We must define our own reference wrapper with an assignment operator.

## Reference Binder

Bind expressions have special handling for `std::reference_wrapper` but knows
nothing about other reference wrappers.
The solution is to mark our reference wrapper as a custom bind expression
to make it interoperable with `std::bind`.

The lazy reference wrapper has an interface similar to `std::reference_wrapper`
but the call operator is used as accessor so it works in bind expressions.

{% highlight c++ %}
namespace lazy {

template <typename T>
struct reference_binder
{
  // Constructors like std::reference_wrapper

  // Lazy call operator

  template <typename... Args>
    requires any_lazy_expression_v<Args...>
  constexpr reference_binder& operator()(Args&&...) {
    return *this;
  }

  // Eager call operator

  template <typename... Args>
    requires (!any_lazy_expression_v<Args...>)
  constexpr T& operator()(Args&&...) {
    return get();
  }

  // Put member operators here
};

} // namespace lazy
{% endhighlight %}

There are two call operators with mutually exclusive constraints.
When bind expressions are evaluated, all nested bind expressions, including
the lazy reference wrapper, are evaluated and replaced by their results.
Nested bind expressions are invoked with the arguments that the bind expression
is invoked with.
These arguments are used by placeholders, not by references.
However, the argument types do indicate whether we are doing an eager or lazy
evaluation which is checked by the constraints.

In eager evaluation we want to use the underlying reference, whereas lazy evaluation
returns the lazy reference wrapper itself as this is necessary for reseating.
Although the reference wrapper itself cannot be reseated, we should be able to
reseat the rest of the bind expression while retaining the lazy reference.

<p class="margintext">
<em>A program may specialize this template for a program-defined type
<tt>T</tt> to have the base characteristics of <tt>true_type</tt> to
indicate that <tt>T</tt> should be treated as a subexpression in a
<tt>bind</tt> call.</em>
<br/>
-- C++ Standard N4950, section [func.bind.isbind]
</p>

The lazy reference wrapper must be registered as a bind expression.

{% highlight c++ %}
namespace std {

template <typename T>
struct is_bind_expression<lazy::reference_binder<T>>
  : true_type {};

} // namespace std {% endhighlight %}

Factory functions for convenience.

{% highlight c++ %}
namespace lazy {

template <typename T>
constexpr auto bind_ref(T& t) -> reference_binder<T> {
  return { t };
}

template <typename T>
void bind_ref(const T&&) = delete;

} // namespace lazy
{% endhighlight %}

The lazy reference wrapper can be used to create bind expressions involving
normal variables.


### Usage

The lazy reference wrapper is a bind expression in the `lazy` namespace,
so it works with preexisting lazy operators that are found by
argument-dependent lookup.

{% highlight c++ %}
auto a_minus_one = bind_ref(a) - 1;

// becomes (via lazy::operator-)
auto a_minus_one = bind(via::minus{}, bind_ref(a), 1);
{% endhighlight %}

Lazy compound assignment can be used to update normal variables.

{% highlight c++ %}
auto increment_a = bind_ref(a) += 1;

// becomes (via lazy::operator+=)
auto increment_a = bind(via::plus_assign, bind_ref(a), 1);
{% endhighlight %}

Lazy functions can be called with lazy reference wrapper arguments.

{% highlight c++ %}
auto ramp_a = via::max(bind_ref(a), 0);

// becomes (via lazy::max)
auto ramp_a = bind(via::max, bind_ref(a), 0);
{% endhighlight %}

Lazy algorithms can also use lazy reference wrappers.
This example adjusts a container such that no element is smaller than `a`.

{% highlight c++ %}
// std::transform(first, last, first, [&a] (auto v) { return std::max(v, a); });
each(1_p = via::max(1_p, bind_ref(a)), first, last);
// where
//   first[0] = std::max(first[0], a)
//   first[1] = std::max(first[1], a)
//   ...
{% endhighlight %}

We can even do fold operations using compound assignment to the referenced variable.
The referenced variable is used to maintain the intermediate folding result.

{% highlight c++ %}
// a = std::accumulate(first, last, a);
each(bind_ref(a) += 1_p, first, last);
{% endhighlight %}

### Member Operators

Some operators only exists as member operators, such as the assignment and
subscript operators.
As an example, we extend the lazy reference wrapper with an assignment operator.

{% highlight c++ %}
namespace lazy {

template <typename T>
struct reference_binder
{
  // ...

  template <typename U>
  constexpr auto operator=(U&& u)
    -> decltype(std::bind(via::assign, *this, declval<U>()))
  {
    return std::bind(via::assign, *this, forward<U>(u));
  }
};

} // namespace lazy
{% endhighlight %}

The example from the introduction works when we use the lazy reference
wrapper with an assignment operator.

{% highlight c++ %}
auto assign_to_a = bind_ref(a) = 1_p;

// becomes (via reference wrapper assignment operator)
auto assign_to_a = bind(via::assign, bind_ref(a), 1_p);
{% endhighlight %}

This assignment bind expression can be evaluated eagerly.

{% highlight c++ %}
assign_to_a(b);

// becomes
bind(via::assign, bind_ref(a), 1_p)(b);

// which becomes
//   bind_ref(a) returns reference to a (via eager call operator)
//   1_p is substituted with b
via:::assign(a, b);

// which becomes
a = b;
{% endhighlight %}

Lazy evaluation and reseating is also possible.

{% highlight c++ %}
auto assign_sum_to_a = assign_to_a(1_p + 2_p);

// becomes
auto assign_sum_to_a = bind(via::assign, bind_ref(a), 1_p)(1_p + 2_p);

// which becomes
//   bind_ref(a) returns bind_ref(a) (via lazy call operator)
//   1_p is substituted with 1_p + 2_p
auto assign_diff_to_a = bind(via::assign, bind_ref(a), bind(plus{}, 1_p, 2_p));
{% endhighlight %}

The reseated assignment expressions can be evaluated eagerly.

{% highlight c++ %}
assign_sum_to_a(b, c);

// becomes
bind(via::assign, bind_ref(a), bind(plus{}, 1_p, 2_p))(b, c);

// which becomes
//   bind_ref(a) returns reference to a (via eager call operator)
//   1_p is substituted with b
//   2_p is substituted with c
via::assign(a, plus{}(b, c));

// which becomes
a = b + c;
{% endhighlight %}

The lazy reference wrapper can be extended with other member operators.
