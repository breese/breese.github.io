---
layout: post
title: Projected Variadic Fold
tags: C++, Template
---

Projected variadic fold is a fold over transformed arguments, and is
functionally equivalent to `std::inner_product` but the input is passed as
function arguments rather than iterators.

<!--more-->

This article is a continuation of [Variadic Fold](https://breese.github.io/2024/08/11/variadic-fold.html),
and will use the same techniques described therein, such as parameter pack
expansion and integer sequences, without further explanation.

## Prologue

While fold is useful for simple cumulative algorithms such as calculating the
sum or the product, other cumulative algorithms are more involved.
An example is the [dot product](https://en.wikipedia.org/wiki/Dot_product),
which exists in the standard library as
[`std::inner_product`](https://en.cppreference.com/w/cpp/algorithm/inner_product).

The dot product of two $N$-dimensional vectors
<p class="eqnarray">
$$\begin{eqnarray*}
\mathbf{a} &=& (a_0, a_1, \ldots, a_{N-1}) \\
\mathbf{b} &=& (b_0, b_1, \ldots, b_{N-1})
\end{eqnarray*}$$
</p>
is given by
<p class="eqnarray">
$$\begin{eqnarray*}
\mathbf{a} \cdot \mathbf{b} &=& \sum_{k=0}^{N-1} a_k\,b_k
\end{eqnarray*}$$
</p>
The elements $a_k$ and $b_k$ are multiplied pair-wise and the results are summed up.

Projected variadic fold is a generalization of the above, where the
addition and multiplication are replaced by arbitrary numeric functions.

{% highlight c++ %}
template <typename F, typename G, typename... Args>
constexpr auto variadic_fold_over(F&& f, G&& g, Args&&... args);
{% endhighlight %}

The algorithm first applies an inner function `g` repeatedly across the
parameter pack, and then folds the results using an outer function `f`.
The first step is called transform or projection, and the second is called
reduction or fold.

Each projection consumes W arguments from the parameter pack, where W is the
[arity](https://en.wikipedia.org/wiki/Arity) of the inner function `g`.
The parameter pack must contain a multiple of W elements.
Using a binary inner function splits the parameter pack into slices with two
elements each.

{% highlight c++ %}
variadic_fold_over(
  f2, // Outer function (fold)
  g2, // Inner function (projection)
  a0, b0, a1, b1, a2, b2);
{% endhighlight %}

becomes the fold of

{% highlight c++ %}
variadic_fold(
  f2,
  g2(a0, b0),
  g2(a1, b1),
  g2(a2, b2));
{% endhighlight %}

which will be expanded as

{% highlight c++ %}
f2(f2(g2(a0, b0),
      g2(a1, b1)),
   g2(a2, b2));
{% endhighlight %}

We assume that the arguments $a_k$ and $b_k$ are interleaved by the user when
calling this algorithm.

The dot product is a projected fold using multiplication as the projection and
addition as the fold.

{% highlight c++ %}
// Dot product
auto result = variadic_fold_over(
  std::plus<>{},
  std::multiplies<>{},
  a0, b0, a1, b1, a2, b2);

assert(result == a0 * b0 + a1 * b1 + a2 * b2);
{% endhighlight %}

Before the implementation of projected fold is explained, we will take a detour
into extended invocation to pick up some useful building-blocks.

## Extended Invocation

The standard library contains [`std::invoke`](https://en.cppreference.com/w/cpp/utility/functional/invoke)
which invokes the given function with the given arguments.
For instance

{% highlight c++ %}
std::invoke(f, x0, x1, x2)
{% endhighlight %}

becomes

{% highlight c++ %}
f(x0, x1, x2)
{% endhighlight %}

### Selective Invocation

Selected invocation invokes the given function with selected arguments determined
by an integer sequence.

Selective invocation is declared as

{% highlight c++ %}
template <typename F, std::size_t... I, typename... Args>
constexpr auto invoke_sequence(F&& f, std::index_sequence<I...>, Args&&... args);
{% endhighlight %}

The integer sequence holds the indices of the arguments to be used.

{% highlight c++ %}
invoke_sequence(f, std::index_sequence<0, 2>{}, x0, x1, x2)
{% endhighlight %}

becomes

{% highlight c++ %}
std::invoke(f, args[0], args[2])
{% endhighlight %}

using C++26 [pack indexing](https://en.cppreference.com/w/cpp/language/pack_indexing),
which amounts to

{% highlight c++ %}
std::invoke(f, x0, x2)
{% endhighlight %}

Selective invocation is fairly easy to implement.
We are not going to use pack indexing because of the C++26 requirement, but
rather we are going to forward the arguments as a tuple to a helper function
that uses `std::get` to select the arguments.

{% highlight c++ %}
template <typename F, std::size_t... I, typename Tuple>
constexpr auto tuple_apply_sequence(F&& f,
                                    std::index_sequence<I...>,
				    Tuple&& tuple) {
  return std::invoke(std::forward<F>(f),
                     std::get<I>(tuple)...);
}

template <typename F, std::size_t... I, typename... Args>
constexpr auto invoke_sequence(F&& f,
                               std::index_sequence<I...> seq,
			       Args&&... args) {
  return tuple_apply_sequence(
    std::forward<F>(f),
    seq,
    std::forward_as_tuple(std::forward<Args>(args)...));
}
{% endhighlight %}

As a side-note, `tuple_apply_sequence` can also be used to implement `std::apply`.

{% highlight c++ %}
template <typename F, typename T>
constexpr auto apply(F&& f, T&& tuple) {
  return tuple_apply_sequence(
    std::forward<F>(f),
    std::make_index_sequence<std::tuple_size<std::remove_cvref_t<T>>::value>{},
    std::forward<T>(tuple));
}
{% endhighlight %}

### Enumerated Integer Sequences

The standard provides functionality to create an integer sequence with the
numbers from $0$ to $N-1$, but we are going to need a facility to create
integer sequences where we can choose the starting value and the step size
between values.

We therefore introduce an extension to create integer sequences that takes 3 arguments
where `N` is the number of integers, `Offset` is the starting value, and
`Step` is the step size. `Offset` defaults to 0, and `Step` defaults to 1.

{% highlight c++ %}
index_sequence_enumerate<N, Offset, Step>
{% endhighlight %}

becomes
{% highlight c++ %}
index_sequence<Offset,
               Offset + Step,
               Offset + Step * 2,
               Offset + Step * 3,
               ...,
               Offset + Step * (N - 1)>
{% endhighlight %}

A couple of concrete examples

{% highlight c++ %}
// Same as std::make_index_sequence<4>
index_sequence_enumerate<4, 0, 1> // becomes index_sequence<0, 1, 2, 3>

// Shifted
index_sequence_enumerate<4, 1, 1> // becomes index_sequence<1, 2, 3, 4>

// Even
index_sequence_enumerate<4, 0, 2> // becomes index_sequence<0, 2, 4, 6>

// Odd
index_sequence_enumerate<4, 1, 2> // becomes index_sequence<1, 3, 5, 7>

// Repeated
index_sequence_enumerate<4, 1, 0> // becomes index_sequence<1, 1, 1, 1>
{% endhighlight %}

The implementation is outside the scope of this article so we just stipulate
that such a facility is available.

### Projected Invocation

Projected invocation is similar to the before-mentioned projected fold, except
the outer function `f` is not a fold function, but just a normal function
that must take as many arguments as produced by the projection.

Projected invocation is declared as

{% highlight c++ %}
template <typename F, typename G, typename... Args>
constexpr auto invoke_over(F&& f, G&& g, Args&&... args);
{% endhighlight %}

If `f3` takes three arguments, and `g2` takes two then

{% highlight c++ %}
invoke_over(f3, g2, a0, b0, a1, b1, a2, b2)
{% endhighlight %}

becomes

{% highlight c++ %}
f3(g2(a0, b0),
   g2(a1, b1),
   g2(a2, b2))
{% endhighlight %}

Projected invocation has to split the parameter pack into smaller slices,
which is done using selective invocation. The intermediate steps can be
illustrated as follows.

{% highlight c++ %}
invoke_over(f3, g2, args...)
{% endhighlight %}

The projection applies `g2` to slices of the parameter pack using generated
integer sequences

{% highlight c++ %}
std::invoke(f3,
            invoke_sequence(g2, index_sequence<0, 1>{}, args...),
            invoke_sequence(g2, index_sequence<2, 3>{}, args...),
            invoke_sequence(g2, index_sequence<4, 5>{}, args...))
{% endhighlight %}

which expands to

{% highlight c++ %}
std::invoke(f3,
            std::invoke(g2, args[0], args[1]),
            std::invoke(g2, args[2], args[3]),
            std::invoke(g2, args[4], args[5]))
{% endhighlight %}

The generic implementation may use a projection function of arbitrary arity,
so the injection of `invoke_sequence` becomes a matter of juggling with integer
sequences to select the proper slices.
While it is possible to determine the arity automatically, we pass it as an
argument below for simplicity.

{% highlight c++ %}
template <typename F, typename G, std::size_t Rank, typename... Args>
constexpr auto invoke_over(F&& f,
                           G&& g,
                           std::integral_constant<std::size_t, Rank> rank,
			   Args&&... args) {
  static_assert((sizeof...(Args) % Rank) == 0,
                "Number of arguments must be a multiple of projection arity");

  return invoke_over_sequence(
    std::forward<F>(f),
    std::forward<G>(g),
    rank,
    // Determine starting offsets of slices.
    // index_sequence<0, Rank, Rank * 2, ..., Rank * (M - 1)>
    // where M is the number of slices (sizeof...(Args) / Rank)
    index_sequence_enumerate<sizeof...(Args) / Rank, 0, Rank>{},
    std::forward<Args>(args)...);
}
{% endhighlight %}

This injects an integer sequence with the starting offsets of the slices and
calls a helper function `invoke_over_sequence` below, which generates the calls
to `invoke_sequence` for each slice.

We start with an integer sequence containing the start of each slice, and the
individual slices will be generated from that.

The helper function is given *two* parameter packs: `Args...` and `Offset...`
which are expanded at different levels, so `Args...` is expanded for each
`Offset`.

{% highlight c++ %}
template <typename F, typename G, typename... Args,
          size_t Rank, size_t... Offset>
constexpr auto invoke_over_sequence(F&& f,
                                    G&& g,
				    std::integral_constant<std::size_t, Rank>,
				    std::index_sequence<Offset...>,
				    Args&&... args) {
  return std::invoke(
    std::forward<F>(f),
    invoke_sequence(
      std::forward<G>(g),
      // The kth offset produces
      // index_sequence<Offset[k],
      //                Offset[k] + 1,
      //                ...,
      //                Offset[k] + (Rank - 1)>
      //
      // If Rank = 3 then Offset... = {0, 3, 6} and thus
      //   k=0: index_sequence<0, 1, 2>
      //   k=1: index_sequence<3, 4, 5>
      //   k=2: index_sequence<6, 7, 8>
      index_sequence_enumerate<Rank, Offset>{},
      std::forward<Args>(args)... // Expands args...
      )... // Expands Offset...
    );
}
{% endhighlight %}

## Projected Fold

Projected fold is a projection invocation where the outer function is a fold.

{% highlight c++ %}
template <typename F, typename G, typename... Args>
constexpr variadic_fold_over(F&& f, G&& g, Args&&... args) {
  return invoke_over(std::bind_front(variadic::fold, std::forward<F>(f)),
                     std::forward<G>(g),
                     std::forward<Args>(args)...);
}
{% endhighlight %}

This creates a function that uses `f` to fold over the results.
The fold uses [partial application](https://en.wikipedia.org/wiki/Partial_application)
in the form of C++20 [`std::bind_front`](https://en.cppreference.com/w/cpp/utility/functional/bind_front)
where

{% highlight c++ %}
bind_front(fold, f)(args...)
{% endhighlight %}

becomes

{% highlight c++ %}
fold(f, args...)
{% endhighlight %}

In order for the binding to work we have changed the
[variadic fold](https://breese.github.io/2024/08/11/variadic-fold.html)
function into a function object.

{% highlight c++ %}
namespace variadic {

inline constexpr struct {
  template <typename F, typename G, typename... Args>
  constexpr auto operator()(F&& f, G&& g, Args&&... args) const {
    return variadic_fold(std::forward<F>(f),
                         std::forward<G>(g),
                         std::forward<Args>(args)...);
  }
} fold{};

} // namespace variadic
{% endhighlight %}

More on function objects in a future article.

This article has used various facilities from newer C++ standards, but these
facilities can be backported to C++11.
