---
layout: post
title: Lazy Evaluation - Automatic Differentiation
tags: C++, Template
---

The derivative of bind expressions can be found with automatic differentiation using
dual numbers.

<!--more-->

## Introduction

[Automatic differentiation](https://en.wikipedia.org/wiki/Automatic_differentiation)
calculates the exact numeric value of the derivative of an expression.
While there are plenty of C++ automatic differentiation frameworks, we are still
going to explore this topic with our own simplified implementation.
This exploration leads to symbolic differentiation of lazy bind expressions.

## Dual Numbers

An elegant approach to automatic differentiation is to use [dual numbers](https://en.wikipedia.org/wiki/Dual_number),
which are [parabolic hypercomplex numbers](https://encyclopediaofmath.org/wiki/Double_and_dual_numbers)
whose squared imaginary unit is zero.

<p class="eqnarray">
$$\begin{eqnarray*}
  x + \epsilon\ \dot{x} \quad\quad\text{where } \epsilon^2 = 0,\ \epsilon \ne 0
\end{eqnarray*}$$
</p>

Dual numbers are a natural choice for differentiation because when they
are used in expressions, the real part calculates the expression and the imaginary
part calculates the derivative of the expression.

Operations on dual numbers can be derived by expanding the equations and
canceling terms with \(\epsilon^2\).
The following shows the result of various operations without the intermediate
derivations.

### Dual Arithmetic

Dual addition corresponds to the [sum rule](https://en.wikipedia.org/wiki/Differentiation_rules#Differentiation_is_linear)

<p class="eqnarray">
$$\begin{eqnarray*}
  \frac{d}{dx} \left(f(x) + g(x)\right) &=& f^\prime(x) + g^\prime(x) & \quad\quad\text{Sum rule} \\ \\
  (x + \epsilon\ \dot{x}) + (y + \epsilon\ \dot{y}) &=& (x + y) + \epsilon\ (\dot{x} + \dot{y}) & \quad\quad\text{Dual addition}
\end{eqnarray*}$$
</p>

Dual multiplication corresponds to the [product rule](https://en.wikipedia.org/wiki/Product_rule)

<p class="eqnarray">
$$\begin{eqnarray*}
  \frac{d}{dx} \left(f(x)\ g(x)\right) &=& f^\prime(x)\ g(x) + f(x)\ g^\prime(x) & \quad\quad\text{Product rule} \\ \\
  (x + \epsilon\ \dot{x}) (y + \epsilon\ \dot{y}) &=& (x y) + \epsilon\ (\dot{x} y + x \dot{y}) & \quad\quad\text{Dual multiplication}
\end{eqnarray*}$$
</p>

Dual division corresponds to the [quotient rule](https://en.wikipedia.org/wiki/Quotient_rule)

<p class="eqnarray">
$$\begin{eqnarray*}
  \frac{d}{dx} \frac{f(x)}{g(x)} &=& \frac{f^\prime(x)\ g(x) - f(x)\ g^\prime(x)}{g(x)^2} & \quad\quad\text{Quotient rule} \\ \\
  \frac{x + \epsilon\ \dot{x}}{y + \epsilon\ \dot{y}} &=& \frac{x}{y} + \epsilon\ \left(\frac{\dot{x} y - x \dot{y}}{y^2}\right) & \quad\quad\text{Dual division}
\end{eqnarray*}$$
</p>

As an example, the dual reciprocal becomes

<p class="eqnarray">
$$\begin{eqnarray*}
  \frac{1 + \epsilon\ 0}{x + \epsilon\ \dot{x}} &=& \frac{1}{x} + \epsilon\ \left(\frac{0 x - 1 \dot{x}}{x^2}\right) \\
  &=& \frac{1}{x} + \epsilon\ \dot{x} \left( - \frac{1}{x^2} \right)
\end{eqnarray*}$$
</p>

### Dual Functions

If we do a [Taylor expansion](https://en.wikipedia.org/wiki/Taylor_series) of an analytic
function `f` with a dual number all higher-order terms cancels out because they contain
an \(\epsilon^2\) which leaves us with

<p class="eqnarray">
$$\begin{eqnarray*}
  f(x + \epsilon\ \dot{x}) &=& f(x) + \epsilon\ \dot{x}\ f^\prime(x) & \quad\quad\text{Dual function}
\end{eqnarray*}$$
</p>

Dual function composition corresponds to the [chain rule](https://en.wikipedia.org/wiki/Chain_rule)

<p class="eqnarray">
$$\begin{eqnarray*}
  \frac{d}{dx} \left(f(g(x))\right) &=& f^\prime(g(x))\ g^\prime(x) & \quad\quad\text{Chain rule} \\ \\
  f(g(x + \epsilon\ \dot{x})) &=& f(g(x)) + \epsilon\ \dot{x}\ f^\prime(g(x))\ g^\prime(x)  & \quad\quad\text{Dual composition}
\end{eqnarray*}$$
</p>


Dual numbers is a fascinating topic, but the above suffices for now.
There are many tutorials if you are interested in further information
about dual numbers.

## Numeric Differentiation

Let us define a dual number class. We refer to the imaginary part as
the *tangent*.

{% highlight c++ %}
namespace dual {

template <typename Value, typename Tangent = Value>
struct number {
  // Types
  using value_type = Value;
  using tangent_type = Tangent;

  // Constructors
  // Assignment operators

  // Accessors
  constexpr value_type value() const;
  constexpr tangent_type tangent() const;
};

} // namespace dual
{% endhighlight %}

Normally dual numbers are defined with just one template parameter that
determines the type of both the real and imaginary parts, similar to
[`std::complex`](https://en.cppreference.com/cpp/numeric/complex).
Later on we need different types, and this is the main reason why we
write our own dual number class in the first place.
The class also needs constructors and assignment operators, but those
are omitted here because they are mostly similar to those of `std::complex`.

We use a convenience function to create a dual number from its arguments.
This is used rather than [class template argument deduction](https://en.cppreference.com/cpp/language/class_template_argument_deduction)
to make it more apparent when we create a new dual number type.

{% highlight c++ %}
namespace dual {

template <typename Value, typename Tangent>
constexpr auto make_number(Value&& value, Tangent&& tangent)
  -> number<remove_cvref_t<Value>, remove_cvref_t<Tangent>> {
  return { forward<Value>(value), forward<Tangent>(tangent) };
}

} // namespace dual
{% endhighlight %}

Assuming we also have an `is_dual_number<T>` type trait that checks if a type is a dual
number, we can overload operators for the dual number class. Only `operator+` is shown.

{% highlight c++ %}
namespace dual {

// Adds two dual numbers
template <typename X0, typename X1, typename Y0, typename Y1>
constexpr auto operator+(const number<X0, X1>& x,
                         const number<Y0, Y1>& y) {
  return make_number(x.value() + y.value(),
                     x.tangent() + y.tangent());
}

// Adds number number with another number type
template <typename X0, typename X1, typename Y>
  requires (!is_dual_number_v<Y>)
constexpr auto operator+(const number<X0, X1>& x,
                         const Y& y) {
  return make_number(x.value() + y,
                     x.tangent());
}

template <typename X, typename Y0, typename Y1>
  requires (!is_dual_number_v<X>)
constexpr auto operator+(const X& x,
                         const number<Y0, Y1>& y) {
  return make_number(x + y.value(),
                     y.tangent());
}

} // namespace dual
{% endhighlight %}

The overloaded operators use `make_number()` to generate the result type.
This ensures that the value and tangent types will be useful when adding
dual numbers of compatible types, e.g. adding `number<int>` and a scalar `float`
results in `number<float>`.
This will also be useful later on when we start doing lazy evaluation.

A production quality implementation would have further constraints to check that
the operations are valid with the given types.
We furthermore assume that all the other operators have been overloaded for dual numbers.

### Partial Derivative

An expression with several dual numbers is a multivariate function.
In this case the tangent arguments are used to determine the
[directional derivative](https://en.wikipedia.org/wiki/Directional_derivative).
The [partial derivative](https://en.wikipedia.org/wiki/Partial_derivative)
is a special case where the directional vector is a unit vector.
Consider a simple equation that is differentiated with respect to \(x\).

<p class="eqnarray">
$$\begin{eqnarray*}
  \frac{\partial}{\partial x} (x^2 + y) &=& 2 x
\end{eqnarray*}$$
</p>

With dual numbers the partial derivative is found by letting
the tangent value of `x` be 1
and all other tangent values be 0.
Assume `a` and `b` are float variables, then we can calculate the
numeric value of the partial derivative as

{% highlight c++ %}
dual::number<float> x{a, 1};
dual::number<float> y{b, 0};

auto r = x * x + y;
// where
//   r.value()   == a * a + b
//   r.tangent() == (a * 1 + 1 * a) + 0 == 2 * a
{% endhighlight %}

### Special Math Functions

We have already seen that applying a dual number to an analytic function
yields an imaginary part that uses the derivative of said function.
Our framework must therefore contain the \(f \to f^\prime\) mapping.
This is usually the bulk of automatic differentiation frameworks, but
we will only demonstrate this with the sine function.

Applying a dual number to the sine function yields

<p class="eqnarray">
$$\begin{eqnarray*}
  \sin(x + \epsilon\ \dot{x}) &=& \sin(x) + \epsilon\ \dot{x} \cos(x)
\end{eqnarray*}$$
</p>

The implementation therefore uses both the sine and the cosine functions.
We start with the customization point objects.

{% highlight c++ %}
namespace via {

inline constexpr struct
{
  template <typename T>
  constexpr auto operator(const T& t) const {
    using std::sin;
    return sin(t);
  }
} sin{};

inline constexpr struct
{
  template <typename T>
  constexpr auto operator(const T& t) const {
    using std::cos;
    return cos(t);
  }
} cos{};

// namespace via
{% endhighlight %}

The dual sine function then becomes

{% highlight c++ %}
namespace dual {

template <typename X0, typename X1>
constexpr auto sin(const number<X0, X1>& x) {
  return make_number(via::sin(x.value()),
                     x.tangent() * via::cos(x.value());
}

} // namespace dual
{% endhighlight %}

The implementation uses `via::sin` and `via::cos` so argument-dependent lookup will find
user-defined overloads of these functions, which we need later for lazy evaluation.

{% highlight c++ %}
dual::number<float> x{a, 1};

auto r = sin(x);
// where
//   r.value()   == std::sin(a)
//   r.tangent() == 1 * std::cos(a)
{% endhighlight %}

### Bind Expression

Bind expressions works seamlessly with dual numbers.
This enables us to calculate the numeric derivative of bind expressions.

{% highlight c++ %}
dual::number<float> x{a, 1};
dual::number<float> y{b, 0};

auto expr = 1_p * 1_p + 2_p;

auto r = expr(x, y);
// where
//   r.value()   == a * a + b
//   r.tangent() == (a * 1 + 1 * a) + 0 == 2 * a
{% endhighlight %}

## Symbolic Differentiation

We can create lazy dual numbers by using lazy placeholders.
Each placeholder has its own type and this is why we defined our dual number
class with two template arguments.

{% highlight c++ %}
auto x = dual::make_number(1_p, 2_p);

// where
//   x.value()   == 1_p
//   x.tangent() == 2_p
{% endhighlight %}

The lazy dual number itself is not a bind expression, but contains two
bind expressions that can be used via the dual number accessors.
This means that when we add two lazy dual numbers, overload resolution finds
`dual::operator+` that we defined above.
This operator does its calculation on the value and tangent element,
which are lazy argument and therefore use `lazy::operator+`.
Their results are combined by `dual::operator+` into a new dual number.

Symbolic differentiation emerges from equations with lazy dual numbers.

{% highlight c++ %}
auto x = dual::make_number(1_p, 1);
auto y = dual::make_number(2_p, 0);

auto expr = x * x + y;

auto f = expr.value();
// where
//   f == 1_p * 1_p + 2_p

auto dfdx = expr.tangent();
// where
//   dfdx == 1 * 1_p + 1_p * 1 + 0
{% endhighlight %}

Eager evaluation yields

{% highlight c++ %}
assert(dfdx(a, b) == 1 * a + a * 1 + 0); // == 2 * a
{% endhighlight %}

### Expression Size

The derived lazy bind expressions can become quite extensive.
The partial derivative of the square operation above yielded
`1_p * 2_p + 2_p * 1_p`, which is \(x \dot{x} + \dot{x} x\) rather than
the reduced \(2 x \dot{x}\).
So the bind expression contains more terms than strictly necessary.
This becomes more apparent for the cube operation.

{% highlight c++ %}
auto expr = x * x * x;

// where
//   expr.value()   == 1_p * 1_p * 1_p
//   expr.tangent() == (1_p * 1_p) * 2_p + (1_p * 2_p + 2_p * 1_p) * 1_p;
{% endhighlight %}

The tangent \((x x) \dot{x} + (x \dot{x} + \dot{x} x) x\) above can be reduced
to \(3 x^2 \dot{x}\) that symbolic differentiation usually yields, but we cannot
reduce the bind expression.
So the resulting bind expression is correct, but large, which increases the
compilation time because the lazy bind expression is encoded by the type
system.

We could reduce the expression size with an exponentiation function instead
of the repeated multiplications above, but that does not solve the general case
with large bind expressions.

### Special Functions

Lazy special functions are also possible due to overload resolution of
lazy functions.

We have already defined a sine function for dual numbers that uses
argument-dependent lookup for both the value and the tangent element,
but we also need lazy overloads for sine and cosine.

{% highlight c++ %}
namespace lazy {

template <typename T>
constexpr auto sin(const T& t) {
  return std::bind(via::sin, t);
}

template <typename T>
constexpr auto cos(const T& t) {
  return std::bind(via::cos, t);
}

} // namespace lazy
{% endhighlight %}

Now we can do symbolic differentiation of the sine function by invoking
it with a lazy dual number.

{% highlight c++ %}
auto x = dual::make_number(1_p, 2_p);
// where
//   x.value()   == 1_p
//   x.tangent() == 2_p
{% endhighlight %}

{% highlight c++ %}
auto f = via::sin(x);

// becomes (via argument-dependent lookup)
auto f = dual::sin(x);

// which becomes
auto f = dual::make_number(via::sin(1_p), 2_p * via::cos(1_p));
{% endhighlight %}

This also works for composed expressions such as \(\sin(x^2)\).

{% highlight c++ %}
auto f = via::sin(x * x);

// becomes (via dual operator overload)
auto f = via::sin(dual::make_number(1_p * 1_p, 2_p * 1_p + 1_p * 2_p));

/// which becomes (via argument-dependent lookup)
auto f = dual::sin(dual::make_number(1_p * 1_p, 2_p * 1_p + 1_p * 2_p));

// which becomes
auto f = dual::make_number(via::sin(1_p * 1_p),
                           (1_p * 2_p + 2_p * 1_p) * via::cos(1_p * 1_p));
{% endhighlight %}

The tangent reduces to \(\dot{x}\ 2 x \cos(x^2)\) as expected by the
chain rule

<p class="eqnarray">
$$\begin{eqnarray*}
  \frac{d}{dx} \sin(x^2) &=& 2\ x \cos(x^2)
\end{eqnarray*}$$
</p>
