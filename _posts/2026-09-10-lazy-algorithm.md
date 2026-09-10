---
layout: post
title: Lazy Evaluation - Algorithms
tags: C++, Template
---

Lazy bind expressions works seamlessly with standard algorithms.
But standard algorithms are limited to univariate or bivariate operations,
whereas bind expressions can take more than two arguments, so we
introduce multivariate for-each and fold algorithms.

<!--more-->

## Introduction

We have already had a glimpse of how lazy bind expressions work with
standard algorithms when we sorted a container according to its absolute
values.
Any algorithm that use a [predicate](https://en.cppreference.com/cpp/named_req/Predicate)
can be used with lazy bind expressions.

{% highlight c++ %}
// Checks for negative elements
auto r = std::any_of(first, last, 1_p < 0);
{% endhighlight %}

{% highlight c++ %}
// Counts positive elements
auto r = std::count_if(first, last, 1_p > 0);
{% endhighlight %}

{% highlight c++ %}
// Counts small elements
auto r = std::count_if(first, last, via::abs(1_p) < threshold);
{% endhighlight %}

{% highlight c++ %}
// Reorders range so negative elements precedes non-negative elements
std::partition(first, last, 1_p < 0);
{% endhighlight %}

## Each

<p class="margintext">
The C++ standard refers to univariate and bivariate operations as unary and binary operations,
but we use the former to avoid confusion with bitwise operations.
</p>
Standard algorithms support univariate and bivariate operations, but bind expressions
can be multivariate operations, i.e. operations that take many arguments.
We are therefore going to implement our own equivalent of `std::for_each`
that iterates in lock-step over many containers.

An arbitrary number of iterators must be passed as a parameter pack, which
has to be at the end of the function argument list, so we put the operation
at the beginning, similar to `std::visit`, and rename the algorithm to `each`.
The first two iterators are the beginning and ending of the first container,
and they determine the number of iterations. This may be followed by an
arbitrary number of begin iterators for other containers that are assumed
to hold at least as many elements as the first container.

{% highlight c++ %}
// Apply multivariate function at each position
inline constexpr struct {
  template <typename F, typename Fwd0, typename... Fwd>
  constexpr void operator()(F&& f, Fwd0 first, Fwd0, last, Fwd... tail) const {
    while (first != last) {
      std::invoke(forward<F>(f), *first++, *tail++...);
    }
  }
} each{};
{% endhighlight %}

Standard algorithms are usually defined as function templates, but `each` is
a function object. We will return to that later.

We should also define an `each_n` algorithm, but this is left as an exercise for
the reader.

### Transformation

The placeholder assignment operator enables us to do 
[transformations](https://en.wikipedia.org/wiki/Map_(higher-order_function))
with the `each` algorithm.

<p class="margintext">
The comment after the algorithm uses subscripting rather than iterator
deferencing and incrementing to make the operations more readable.
</p>
Univariate transformation can be done as.

{% highlight c++ %}
// std::transform(first, last, output, negate{});
each(2_p = -1_p, first, last, output);
// where
//  output[0] = -first[0]
//  output[1] = -first[1]
//  ...
{% endhighlight %}

The bind expression determines where the results are written, so in-place
transformation can be done by assigning the result to the current element.

{% highlight c++ %}
// std::transform(first, last, first, [] (auto a) { return std::abs(a); });
each(1_p = via::abs(1_p), first, last);
// where
//   first[0] = via::abs(first[0])
//   first[1] = via::abs(first[1])
//   ...
{% endhighlight %}

Bivariate transformation is a pair-wise operation at each position in
two containers.
Notice that `last` cannot be accessed by the bind expression so it has
no associated placeholder.
The placeholders are associated with the begin iterators of the various containers.

{% highlight c++ %}
// std::transform(first, last, second, output, plus{});
each(3_p = 1_p + 2_p, first, last, second, output);
// where
//   output[0] = first[0] + second[0]
//   output[1] = first[1] + second[1]
//   ...
{% endhighlight %}

In-place transformation can be done with compound assignment.

{% highlight c++ %}
// std::transform(first, last, second, first, plus{});
each(1_p += 2_p, first, last, second);
// where
//   first[0] += second[0]
//   first[1] += second[1]
//   ...
{% endhighlight %}

Multivariate transformation is an element-wise operation at each position
of all containers.

{% highlight c++ %}
// AXPY
each(1_p = 2_p * 3_p + 4_p,
     begin(output), end(output), begin(va), begin(vx), begin(vy));
// where
//   output[0] = va[0] * vx[0] + vy[0]
//   output[1] = va[1] * vx[1] + vy[1]
//   ...
{% endhighlight %}

## Fold

[Folding](https://en.wikipedia.org/wiki/Fold_(higher-order_function)) iterates
in lock-step over many containers, but also maintains an intermediate result
that is used in each operation.

{% highlight c++ %}
// Apply multivariate function at each position
inline constexpr struct {
  template <typename F, typename T, typename Fwd0, typename... Fwd>
  constexpr T operator()(F&& f,
                         T init,
                         Fwd0 first, Fwd0, last, Fwd... tail) const {
    while (first != last) {
      init = std::invoke(forward<F>(f), init, *first++, *tail++...);
    }
    return init;
  }
} fold{};
{% endhighlight %}

This is a left fold that can be expressed as

<p class="eqnarray">
$$\begin{eqnarray*}
  y &=& f(\ldots f(f(init, x_0), x_1), \ldots x_n)
\end{eqnarray*}$$
</p>

The initial value of the intermediate variable is passed as argument, just like
in standard numeric algorithms.
This is usually set to the [neutral element](https://en.wikipedia.org/wiki/Identity_element)
of the operation `f`.

The intermediate variable is accessed with the first placeholder `1_p`.

### Folding One Container

The sum is found by adding all container elements.

{% highlight c++ %}
// auto sum = std::accumulate(first, last, 0);
auto sum = fold(1_p + 2_p, 0, first, last);
// where
//   sum = 0
//   sum = sum + first[0]
//   sum = sum + first[1]
//   ...
{% endhighlight %}

The product is found by multiplying all container elements using the
multiplicative identity 1 as the initial value.

{% highlight c++ %}
// auto product = std::accumulate(first, last, 1, multiplies{});
auto product = fold(1_p * 2_p, 1, first, last);
// where
//   product = 1
//   product = product * first[0]
//   product = product * first[1]
//   ...
{% endhighlight %}

We can also fold with [lazy functions]({% post_url 2026-09-06-lazy-function %}).
The lazy functions may have different neutral elements.

{% highlight c++ %}
auto r = fold(via::max(1_p, 2_p), numeric_limits<T>::min(), first, last);
// where
//   r = numeric_limits<T>::min()
//   r = std::max(r, first[0])
//   r = std::max(r, first[1])
//   ...
{% endhighlight %}

A more elaborate example is the [mean absolute error](https://en.wikipedia.org/wiki/Mean_absolute_error)

<p class="eqnarray">
$$\begin{eqnarray*}
  r &=& \frac{1}{N} \sum_i^N | x_i - \bar{x} |
\end{eqnarray*}$$
</p>
where $\bar{x}$ is the mean value of the container.

The mean must be calculated first and then passed as an argument to the above equation.

{% highlight c++ %}
auto mean = fold(1_p + 2_p,
                 0, begin(a), end(a)) / size(a);
auto mae  = fold(1_p + via::abs(2_p - mean),
                 0, begin(a), end(a)) / size(a);
{% endhighlight %}

The [mean squared error](https://en.wikipedia.org/wiki/Mean_squared_error)
is done almost identically.

<p class="eqnarray">
$$\begin{eqnarray*}
  r &=& \frac{1}{N} \sum_i^N ( x_i - \bar{x})^2
\end{eqnarray*}$$
</p>

We do not have a lazy square function, but
can easily create a bind expression for it.

{% highlight c++ %}
// mean as above
auto square = 1_p * 1_p;
auto mse    = fold(1_p  + square(2_p - mean),
                   0, begin(a), end(a)) / size(a);
{% endhighlight %}

The same can be done with direct reseating of the square function.

{% highlight c++ %}
// mean as above
auto mse  = fold(1_p  + (1_p * 1_p)(2_p - mean),
                 0, begin(a), end(a)) / size(a);
{% endhighlight %}

There is a subtlety to consider when reading the bind expression above.
The first occurrence of placeholder `1_p` is not the same as the two used for
squaring. The former applies to the accumulation expression and will be
substituted by data, whereas the latter applies to the reseating expression
and will be substituted by the reseating argument.

{% highlight c++ %}
auto expr = 1_p  + (1_p * 1_p)(2_p - mean);

// becomes (via reseating)
auto expr = 1_p + (2_p - mean) * (2_p - mean);
{% endhighlight %}

### Folding Two Containers

The dot product is a sum of pair-wise multiplications of elements of
two containers.

<p class="eqnarray">
$$\begin{eqnarray*}
  r &=& \sum_i x_i\ y_i
\end{eqnarray*}$$
</p>

This can be computed by

{% highlight c++ %}
// int sum = std::inner_product(first, last, second, 0);
auto r = fold(1_p + 2_p * 3_p, 0, first, last, second);
// where
//   r = 0
//   r = r + first[0] * second[0]
//   r = r + first[1] * second[1]
//   ...
{% endhighlight %}

`std::inner_product` allows both the element-wise multiplication and the
summation to be changed.
Technically speaking the new computation is not the dot product anymore,
but it is still a fold over two containers.

{% highlight c++ %}
// auto r = std::inner_product(first, last, second, 1,
//                             multiplies{}, minus{});
auto r = fold(1_p *= 2_p - 3_p, 1, first, last, second);
// where
//   r = 1
//   r = r * (first[0] - second[0])
//   r = r * (first[1] - second[1])
//   ...
{% endhighlight %}

Multivariate folding is a simple extension of the above. Just add more
containers and more placeholders.

## Partial Fold

The partial fold is a fold operation that writes the intermediate results
to an output container.
This can be done using the `fold` algorithm if the bind expression assigns
to the output container.

{% highlight c++ %}
// std::partial_sum(first, last, output)
auto r = fold(3_p = 1_p + 2_p, 0, first, last, output);
// where
//   output[0] = init + first[0]
//   output[1] = output[0] + first[1]
//   output[2] = output[1] + first[2]
//   ...
{% endhighlight %}

<p class="margintext">
<em>The assignment operator $($=$)$ and the compound assignment operators all
group left-to-right. [...] their result is an lvalue</em>
<br/>
-- C++ Standard N4950, section [expr.ass]
</p>
The `fold` algorithm updates the intermediate variable `1_p` so the above
assignment expression is equivalent to `1_p = (3_p = 1_p + 2_p)` which assigns
to both `1_p` and `3_p`.
This is possible because an assignment is an expression that can be part of
another expression, including another assignment expression.
This is why assignment and compound assignment operators usually return `*this`.

The in-place partial sum is simply a matter of updating the elements of the
input container.

{% highlight c++ %}
// std::partial_sum(first, last, first)
auto r = fold(2_p += 1_p, 0, first, last);
// where
//   first[0] += init
//   first[1] += first[0]
//   first[2] += first[1]
//   ...
{% endhighlight %}

The partial product is a trivial change of the partial sum.

{% highlight c++ %}
// std::partial_sum(first, last, output, multiplies{})
auto r = fold(3_p = 1_p * 2_p, 1, first, last, output);
// where
//   output[0] = init * first[0]
//   output[1] = output[0] * first[1]
//   output[2] = output[1] * first[2]
//   ...
{% endhighlight %}

We can even calculate the partial dot product with a more elaborate bind expression.

{% highlight c++ %}
auto r = fold(4_p = 1_p + 2_p * 3_p, 0, first, last, second, output);
// where
//   output[0] = init + first[0] * second[0]
//   output[1] = output[0] + first[1] * second[1]
//   output[2] = output[1] + first[2] * second[2]
//   ...
{% endhighlight %}

## Tacit Programming

<p class="margintext">
<em>Tacit programming, also called point-free style, refers to usage of
tacit functions that are defined in terms of implicit arguments [...]
This allows creating [compound] derived functions without specifying
any arguments explicitly.</em>
<br/>
-- <a href="https://aplwiki.com/wiki/Tacit_programming">APL Wiki</a>
</p>
So far we have reimplemented various standard algorithms using the `each` and
`fold` algorithms with specific bind expressions.
We can give these algorithms their own names.

`each` is a function object whose first argument is the function it applies,
so we can use partial application to create named algorithms.

{% highlight c++ %}
auto axpy = std::bind_front(each, 1_p = 2_p * 3_p + 4_p);
{% endhighlight %}

{% highlight c++ %}
axpy(begin(result), end(result), begin(va), begin(vx), begin(vy));

// becomes (via bind_front)
each(1_p = 2_p * 3_p + 4_p,
     begin(result), end(result), begin(va), begin(vx), begin(vy))
{% endhighlight %}

The same can be done with `fold`.

{% highlight c++ %}
auto dot_product = std::bind_front(fold, 1_p += 2_p * 3_p);
{% endhighlight %}

{% highlight c++ %}
// auto r = std::inner_product(first, last, second, 0);
auto r = dot_product(0, first, last, second);

// becomes (via bind_front)
auto r = fold(1_p += 2_p * 3_p, 0, first, last, second);
{% endhighlight %}
