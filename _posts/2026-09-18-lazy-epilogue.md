---
layout: post
title: Lazy Evaluation - Epilogue
tags: C++, Template
---

Summary of lazy evaluation using bind expressions.

<!--more-->

This series on lazy evaluation as unveiled an untapped potential of bind expressions.
Lazy bind expressions provide a compact notation for their construction, which has
many applications including symbolic differentiation.
Lazy bind expressions are compliant with the standard.

### Prologue

The [prologue]({% post_url 2026-09-03-lazy-prologue %}) explains bind expressions
and how to pass overloaded functions as function arguments by wrapping them as
function objects.

### Lazy Operators

[Lazy operators]({% post_url 2026-09-04-lazy-operator %}) provides a compact
notation to create bind expressions.
This is done by overloading operators for lazy arguments -- bind expressions
or placeholders.

A custom placeholder with an assignment operator is introduced.
This placeholder is used as a user-defined literal, such as `1_p`.

{% highlight c++ %}
sort(first, last, 1_p < 2_p);
{% endhighlight c++ %}

### Lazy Functions

[Lazy functions]({% post_url 2026-09-06-lazy-function %}) supplements the compact
notation with overloaded functions for lazy arguments.

|![Reseat ramp](/assets/lazy/reseat-diff.png){: style="width:100%;"}|
|Extending at placeholder|
{: class="marginimage"}

{% highlight c++ %}
sort(first, last, via::abs(1_p) < via::abs(2_p));
{% endhighlight c++ %}

The lazy bind expressions uses customization point objects as the bound function.
These are function objects that use argument-dependent lookup to find overloaded
functions.
As a consequence, lazy functions can be invoked with lazy arguments to produce
new lazy bind expressions.

{% highlight c++ %}
auto ramp      = via::max(0, 1_p); // Bind expression
auto ramp_diff = ramp(1_p - 2_p);  // Extended bind expression
{% endhighlight c++ %}

### Lazy Algorithms

[Lazy algorithms]({% post_url 2026-09-10-lazy-algorithm %}) demonstrates
lazy bind expressions working seamlessly with standard algorithms.

{% highlight c++ %}
// Counts positive elements
auto r = std::count_if(first, last, 1_p > 0);
{% endhighlight %}

Multivariate for-each and fold algorithms are provided.

{% highlight c++ %}
// int sum = std::inner_product(first, last, second, 0);

auto r = fold(1_p + 2_p * 3_p, 0, first, last, second);
// where
//   r = 0
//   r = r + first[0] * second[0]
//   r = r + first[1] * second[1]
//   ...
{% endhighlight %}

### Lazy References

[Lazy references]({% post_url 2026-09-12-lazy-reference %})
are reference wrappers with member operators, such as assignment or
subscripting.

{% highlight c++ %}
// a = std::accumulate(first, last, a);

each(bind_ref(a) += 1_p, first, last);
{% endhighlight %}

### Expression Templates

[Expression templates]({% post_url 2026-09-15-lazy-expression-template %})
are syntactic sugar for element-wise vector operations.
Lazy references and bind expressions are used as building-blocks.

### Automatic Differentiation

[Automatic differentiation]({% post_url 2026-09-17-lazy-autodiff %}) is
possible by combining lazy bind expressions with dual numbers.
Symbolic differentiation materializes when using lazy dual numbers.

Consider the derivative of a composed function

<p class="eqnarray">
$$\begin{eqnarray*}
  \frac{d}{dx} \sin(x^2) &=& 2\ x \cos(x^2)
\end{eqnarray*}$$
</p>

Invoking the function with a lazy dual number yields its derivative as a bind expression.

{% highlight c++ %}
auto x = dual::make_number(1_p, 1);

auto f = via::sin(x * x);
// where
//   f.tangent() == (1_p * 1 + 1 * 1_p) * via::cos(1_p * 1_p);
{% endhighlight %}
