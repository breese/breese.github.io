---
layout: post
title: Lazy Evaluation - Expression Templates
tags: C++, Template
---

Expression templates can be implemented with lazy bind expressions.

<!--more-->

<p class="margintext">
Todd Veldhuizen, "Expression Templates", C++ Report 7(5), pp. 26-31, 1995.
</p>
Expression templates provides a compact syntax for element-wise vector operations
with an efficient implementation.
Expression template frameworks usually have their own expression types, but we
shall see below that bind expressions can be used instead.

<p class="margintext">
Hoemmen, Hollman and Trott, "Evolving a Standard C++ Linear Algebra Library from the BLAS", <a href="https://wg21.link/P1674">P1674</a>, 2022.
</p>
The purpose is to demonstrate the usefulness of bind expressions, not necessarily
to promote the use of expression templates. See P1674 for a balanced account
of expression templates and its alternatives.

### Introduction

Suppose we have an N-dimensional numeric vector class with an overloaded operator
for element-wise addition.

{% highlight c++ %}
template <typename T, size_t N>
struct vec;

template <typename T, size_t N>
constexpr vec<T, N> operator+(const vec<T, N>&, const vec<T, N>&);
{% endhighlight %}

Assume we have three numeric vector object `va`, `vb`, and `vc`.
Adding these three vectors can be done with the overloaded operator.
{% highlight c++ %}
vec<T, N> r = va + vb + vc;
{% endhighlight %}

A na&iuml;ve implementation of the operator has a loop that does pairwise addition.
The above equation with two additions results in two loops.
We would like a single loop to perform all additions at once to improve efficiency.
Some compilers may be able to merge the loops automatically using an
optimization technique called [loop fusion](https://en.wikipedia.org/wiki/Loop_fission_and_fusion),
but if we want guarantees then we need to help the compiler.
This motivates expression templates.

## Expression Templates

The basic trick of expression templates is that the overloaded operators
collects the operations into an expression and the vector constructor applies
the collected expression to each element in the containers using a single loop.

We need two overloaded operators: one to add two vectors, and another to
extend an expression by adding yet another vector.
{% highlight c++ %}
{% assign vecexpr="expression" %}
// {{vecexpr}} is defined later

template <typename T, size_t N>
constexpr {{vecexpr}} operator+(const vec<T, N>& lhs, const vec<T, N>& rhs)

template <size_t N>
constexpr {{vecexpr}} operator+({{vecexpr}} lhs, const vec<N>& rhs);

template <typename T, size_t N>
struct vec {
  vec({{vecexpr}}); // Single loop with element-wise operations
};
{% endhighlight %}

Our earlier example is processed as follows. We deliberately use a vague
and syntactically incorrect notation for expressions below, because we
want to convey an idea without drowning in detail.
{% highlight c++ %}
vec<T, N> r = va + vb + vc;

// becomes (via first operator+)
vec<T, N> r = expression(+, va, vb) + vc;

// which becomes (via second operator+)
vec<T, N> r = expression(+, expression(+, va, vb), vc);
{% endhighlight %}

The left-hand side uses the vector type rather than automatic type deduction
(`auto r = ...`)
because the latter would result in an expression without constructing
a resulting vector.

The expression above looks remarkably like a nested bind expression.
However, we need a bind expression that captures containers by reference
and that can index the container elements.
We return to that in a moment.

The constructor that takes the bind expression can be written as

{% highlight c++ %}
template <typename T, size_t N>
struct vec : array<T, N> {

  // Apply collected expression at each position.
  template <typename Expr>
    requires is_bind_expression_v<remove_cvref_t<Expr>>
  vec(Expr&& expr) {
    for (size_t k = 0; k < N; ++k) {
      (*this)[k] = expr(k);
    }
  }
};
{% endhighlight %}

The expression uses the call operator rather than the subscript operator
for indexing. That is simply how bind expressions work.

Other constructors are needed to initialize the vector with values, but we
omit these for brevity.

## Subscriptable Reference

The collected expression must remember which containers are used where in the expression,
so they are captured by reference into the bind expression to prevent unnecesary copying.
The bind expression must also specify which index is used to access container elements.
This is done by extending the [lazy reference wrapper]({% post_url 2026-09-12-lazy-reference %})
with a subscript operator.

<p class="margintext">
Peter Dimov and Barry Revzin, "Adding functionality to placeholder types", <a href="https://wg21.link/P3171">P3171</a>, 2024.
</p>
First we need a function object that wraps the subscript operator as
proposed by P3171.

{% highlight c++ %}
namespace via {

inline constexpr struct {
  template <typename Lhs, typename Rhs>
  constexpr auto operator()(Lhs&& lhs, Rhs&& rhs) const
    -> decltype(declval<Lhs>()[declval<Rhs>()])
  {
    return forward<Lhs>(lhs)[forward<Rhs>(rhs)];
  }
} subscript{};

} // namespace via
{% endhighlight %}

As a reminder, the trailing return type is used to constrain the call operator.

The subscript operator for the lazy reference wrapper uses this function object.

{% highlight c++ %}
namespace lazy {

template <typename T>
struct reference_binder
{
  // ...

  template <typename U>
  constexpr auto operator[](U&& u) const {
    return std::bind(via::subscript, *this, forward<U>(u));
  }
};

} // namespace lazy
{% endhighlight %}

We can now create subscriptable bind expressions.

{% highlight c++ %}
auto va_at = bind_ref(va)[1_p];

// becomes (via reference_binder<T>::operator[])
auto va_at = bind(via::subscript, bind_ref(va), 1_p);
{% endhighlight %}

Eager evaluation yields

{% highlight c++ %}
auto r = va_at(k);

// becomes
auto r = bind(via::subscript, bind_ref(va), 1_p)(k);

// which becomes
//   bind_ref(va) becomes reference to va (via eager call operator)
//   1_p is substituted with k
auto r = via::subscript(va, k);

// which becomes
auto r = va[k];
{% endhighlight %}

### Vector Operators

The overloaded `operator+` for the numeric vector returns a bind expression
that can be invoked with an index as the first argument, represented by `1_p`.

{% highlight c++ %}
template <typename T, size_t N>
constexpr auto operator+(const vec<T, N>& lhs, const vec<T, N>& rhs) {
  return bind_ref(lhs)[1_p] + bind_ref(rhs)[1_p);
}

template <typename Lhs, typename N, size_t N>
  requires is_bind_expression_v<Lhs>
constexpr auto operator+(Lhs&& lhs, const vec<T, N>& rhs) {
  return forward<Lhs>(lhs) + bind_ref(rhs)[1_p];
}
{% endhighlight %}

Adding three vectors creates a nested bind expression.
{% highlight c++ %}
vec<T, N> r = va + vb + vc;

// becomes (via first vec operator+)
vec<T, N> r = (bind_ref(va)[1_p] + bind_ref(vb)[1_p]) + vc;

// which becomes (via lazy::operator+)
vec<T, N> r = bind(plus{}, bind_ref(va)[1_p], bind_ref(vb)[1_p])) + vc;

// which becomes (via second vec operator+)
vec<T, N> r = bind(plus{}, bind_ref(va)[1_p], bind_ref(vb)[1_p]))
              + bind_ref(vc)[1_p];

// which becomes (via lazy::operator+)
vec<T, N> r = bind(plus{},
                   bind(plus{}, bind_ref(va)[1_p], bind_ref(vb)[1_p]),
                   bind_ref(vc)[1_p]);
{% endhighlight %}

The constructor then expands into the desired expression.

{% highlight c++ %}
for (size_t k = 0; k < N; ++k) {
  r[k] = bind(plus{},
              bind(plus{}, bind_ref(va)[1_p], bind_ref(vb)[1_p]),
              bind_ref(vc)[1_p])(k);
}

// becomes
for (size_t k = 0; k < N; ++k) {
  r[k] = plus{}(plus{}(va[k], vb[k]), vc[k]);
}

// which becomes
for (size_t k = 0; k < N; ++k) {
  r[k] = (va[k] + vb[k]) + vc[k];
}
{% endhighlight %}

Adding the remaining overloaded operators and functions gives us an expression
template framework based on bind expressions.
