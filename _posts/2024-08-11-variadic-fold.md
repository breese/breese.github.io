---
layout: post
title: Variadic Fold
tags: C++, Template
---

Variadic fold and partial fold are generalizations of
[fold expressions](https://en.cppreference.com/w/cpp/language/fold).
Variadic partial fold produces the intermediate results of a fold.
Fold expressions were added to the language in C++17, whereas variadic fold works
from C++11 where parameter packs were added.

Folding is useful for efficient numeric computation that requires loop unrolling
even when the compiler is optimizing for minimal size.

<!--more-->

## Prologue

Fold expressions are limited to operators.
For instance, `operator+` is used when accumulating a set of arguments

{% highlight c++ %}
template <typename T, typename... Args>
constexpr auto foldexpr_accumulate(T init, Args... args) {
  return (init + ... + args);
}
{% endhighlight %}

Variadic fold works with [callables](https://en.cppreference.com/w/cpp/named_req/Callable),
just like the iterator-based standard numeric algorithms.
Accumulation is done with [`std::plus`](https://en.cppreference.com/w/cpp/utility/functional/plus)
which is a function object that invokes `operator+`

{% highlight c++ %}
template <typename T, typename... Args>
constexpr auto variadic_accumulate(T init, Args... args) {
  return variadic_fold(std::plus<>{}, init, args...);
}
{% endhighlight %}

Folding an arbitrary function with fold expressions requires an overloaded
operator.
Variadic fold can use the function directly, or indirectly
via a function object to handle overloaded functions.

## Folding

[Fold](https://en.wikipedia.org/wiki/Fold_(higher-order_function)) is the
recurrent application of a function to a sequence of arguments.
Given an operator $\otimes$ and arguments $x_0, \ldots, x_3$, a left-fold
expression can be expanded with infix notation as
<p class="eqnarray">
$$\begin{eqnarray*}
y &=& ((x_0 \otimes x_1) \otimes x_2) \otimes x_3
\end{eqnarray*}$$
</p>

Equivalently, given a function $f$ a variadic left-fold can be expanded with
prefix notation as
<p class="eqnarray">
$$\begin{eqnarray*}
y &=& f(f(f(x_0, x_1), x_2), x_3) \\
\end{eqnarray*}$$
</p>

## Variadic Fold

Variadic fold is declared to take a function `f` and an parameter pack `args...`

{% highlight c++ %}
template <typename F, typename... Args>
constexpr auto variadic_fold(F&& f, Args&&... args);
{% endhighlight %}

The basic idea is to use parameter pack expansion to traverse the arguments.

### Parameter Pack

<p class="margintext">
<em>A template parameter pack is a template parameter that accepts zero or more template arguments.
<br/>
A function parameter pack is a function parameter that accepts zero or more function arguments.
<br/>
A pack expansion consists of a pattern and an ellipsis, the initialization of which produces
zero or more instantiations of the pattern in a list.
</em>
<br/>
C++ Standard, section $[$temp.variadic$]$
</p>

A [parameter pack](https://en.cppreference.com/w/cpp/language/parameter_pack) can be regarded
as a simple language-level tuple for function parameters.
Expanding a parameter pack will generate a sequence of operations.

Initially function `f` should be invoked with the first two arguments of the parameter pack.
Subsequent invocations of `f` should take the previous result and the next argument, that
is, the $k$th invocation should be `f(partial[k-1], args[k])` where `partial[k-1]`
is the result of the $k-1$th invocation.

This can be done by maintaining an array of partial results. The parameter pack is
then expanded by the [brace-enclosed initializer](https://en.cppreference.com/w/cpp/language/list_initialization)
to ensure left-to-right invocation.
The implementation below is not correct, but brings us closer to a solution.

{% highlight c++ %}
// Notice: Use of previous is not correct yet.
// Notice: Perfect forwarding omitted for brevity.

template <typename F, typename T, typename... Args>
constexpr auto variadic_fold(F f, T init, Args... args) {
  using result_type = std::common_type_t<T, Args...>;
  // Expand parameter pack in brace-enclosed initialization
  result_type partial[]{
    init,
    f(partial[previous], args)... };

  // Result of last invocation
  return partial[sizeof...(Args)];
}
{% endhighlight %}

The local array should contain the following entries.

{% highlight c++ %}
  result_type partial[]{
    init,
    f(partial[0], args[0]),
    f(partial[1], args[1]),
    ...
    f(partial[N-1], args[N-1]) };
{% endhighlight %}

Notice that the array initializer uses previously initialized elements
`partial[k-1]` so we only access already-initialized elements.
C++26 [pack indexing](https://en.cppreference.com/w/cpp/language/pack_indexing)
of the arguments is used merely for illustrative purposes.
In practice we use parameter pack expansion for the arguments.

The `partial[k]` result on left-hand side of the function is using indices from
$0$ to $N-1$ which we will get from an integer sequence.

### Integer Sequence

<p class="margintext">
<em>$[$A$]$ class template that can represent an integer sequence. When used as an argument to a
function template the template parameter pack defining the sequence can be deduced and used
in a pack expansion.</em>
<br/>
C++ Standard, section $[$intseq.general$]$/1
</p>

We have to create a new parameter pack containing the indices of the previous
partial results.
This is done in two steps: first generate an integer sequence, and then match
this integer sequence in a template to obtain the parameter pack.

{% highlight c++ %}
// Example to demonstrate the deduction of a parameter pack from an integer
// sequence

template <typename> struct iota;

template <std::size_t... Indices>
struct iota<std::index_sequence<Indices...>> // Step 2: Deduces parameter pack
{
  std::size_t array[sizeof...(Indices)]{ Indices... }; // Expands parameter pack
};

// Step 1: Generates index_sequence<0, 1, ..., N-1>
template <size_t N>
using iota_n = iota<std::make_index_sequence<N>>;

// Usage

constexpr auto iota_2 = iota_n<2>{};

static_assert(iota_2.array[0] == 0);
static_assert(iota_2.array[1] == 1);
{% endhighlight %}

### C++14 Variadic Fold

The variadic fold algorithm creates an integer sequence and calls a helper
function denoted `variadic_fold_sequence` below. The helper function has *two* parameter
packs: `Indices...` and `Args...` which are expanded pair-wise.

{% highlight c++ %}
template <std::size_t... Indices, typename F, typename T, typename... Args>
constexpr auto variadic_fold_sequence(std::index_sequence<Indices...>,
                                      F f,
                                      T init,
                                      Args... args) {
  using result_type = std::common_type_t<T, Args...>;
  // Pair-wise parameter pack expansion
  result_type partial[]{ init, f(partial[Indices], args)... };

  // Result of last invocation
  return partial[sizeof...(Args)];
}

template <typename F, typename T, typename... Args>
constexpr auto variadic_fold(F&& f, T&& init, Args&&... args) {
  return variadic_fold_sequence(
    std::index_sequence_for<Args...>{}, // Inject indices
    std::forward<F>(f),
    std::forward<T>(init),
    std::forward<Args>(args)...);
}
{% endhighlight %}

### C++11 Variadic Fold

The above works for C++14, but it is possible to backport this to C++11.
The trick is to replace the `variadic_fold_sequence` function with a class that
has the `partial` array as a member variable, and then do all the work in the
constructor.

{% highlight c++ %}
// Variadic fold with C++11 (assuming we have backported index_sequence)

template <typename, typename>
struct variadic_fold_sequence;

template <typename T, std::size_t... Indices>
struct variadic_fold_sequence<T, index_sequence<Indices...>> {
  template <typename F, typename... Args>
  constexpr variadic_fold_sequence(F&& f,
                                   T init,
                                   Args... args)
    // Pair-wise parameter pack expansion
    : partial{ init, f(partial[Indices], args)... } {
  }

  T partial[1 + sizeof...(Indices)];
};

template <typename F, typename T, typename... Args,
          typename R = typename std::common_type<T, Args...>::type>
constexpr R variadic_fold(F&& f, T&& init, Args&&... args) {
  // Create temporary and return its final result.
  return variadic_fold_sequence<R, index_sequence_for<Args...>> // Inject indices
    (std::forward<F>(f),
     std::forward<T>(init),
     std::forward<Args>(args)...)
    .partial[sizeof...(Args)];   // Result of last invocation
}
{% endhighlight %}

Other minutiae are needed for a production-ready variadic fold, such as using
`std::invoke` rather than calling `f` directly, but we omit these here.

## Variadic Partial Fold

Variadic partial fold is a generalization of variadic fold that returns the
intermediate results of the fold.

Variadic partial fold is functionally equivalent to
[`std::partial_sum`](https://en.cppreference.com/w/cpp/algorithm/partial_sum),
but uses function parameters rather than iterators.
Fold expressions do not support partial fold.

We saw in the implementation of variadic fold how to create a temporary array
holding the intermediate results, so it ought to be simple to design a variadic
partial fold algorithms. There are, however, some complications that we need
to address.

The intermediate results are returned by writing to a sequence of output
parameters.
For simplicity we will store the intermediate results in-place, so the input
parameters are also the output parameters.

Variadic partial fold is declared to take a function `f` and an parameter pack `args...`
of mutable lvalue-references

{% highlight c++ %}
template <typename F, typename... Args>
constexpr auto variadic_partial_fold(F&& f, Args&... args);
{% endhighlight %}

We would expect the variadic partial fold to work as follows:

{% highlight c++ %}
// Using variables
int a = 11;
int b = 22;
int c = 33;
variadic_partial_fold(a, b, c);
assert(a == 11);
assert(b == a + 22);
assert(c == b + 33);

// Using an array
int values[]{ 11, 22, 33 };
variadic_partial_fold(values[0], values[1], values[2]);
assert(values[0] == 11);
assert(values[1] == values[0] + 22);
assert(values[2] == values[1] + 33);
{% endhighlight %}

### Left-to-Right Evaluation

Variadic partial fold must evaluate the intermediate results in a left-to-right
manner, and it must be able to access the previous result.

In the variadic fold implementation the intermediate results were stored in a
temporary array and the previous result was obtained by accessing this array.
Variadic partial fold stores the intermediate results in the output parameters,
not in an array, so we need different way of accessing the previous result.

Furthermore, the arguments need not have the same type.
It is entirely possible to do both variadic fold and partial fold over a
combination of, say, integral and floating-point numbers, as long as the given
function `f` can handle these combinations.

With that in mind, we choose to forward the parameters as a tuple and access
the previous result using [`std::get`](https://en.cppreference.com/w/cpp/utility/tuple/get).

For the left-to-right evaluation we still need a temporary array in order to
use brace-enclosed initialization as before.
The intermediate results are assigned to the output parameters as a side-effect
of initializing the array.
The sole purpose of the temporary array is to ensure left-to-right evaluation.

The assignment returns a reference that we do not need, so the array should
discard these references.
One possibility is to use the comma expression trick
{% highlight c++ %}
// Assignment is a side-effect
using expander = int[];
expander{ (void(a = 11), 0), (void(b = a + 22), 0) };
{% endhighlight %}
Another possibility is to introduce an empty helper class that ignores all
arguments passed during construction.

{% highlight c++ %}
struct discard
{
    template <typename T>
    constexpr discard(T&&) {}
};

// Assigment is a side-effect
using expander = discard[];
expander{ a = 11, b = a + 22 };
{% endhighlight %}

### C++11 Variadic Partial Fold

Now we just need to assemble all the parts.
The variadic partial fold injects an integer sequence and uses a helper class
with a member array to expand the partial fold assignments.

{% highlight c++ %}
// Variadic partial fold with C++11 (assuming we have backported index_sequence)

template <typename>
struct variadic_partial_fold_sequence;

template <std::size_t... Indices>
struct variadic_partial_fold_sequence<index_sequence<Indices...>> {
  template <typename F, typename Tuple>
  constexpr variadic_partial_fold_sequence(F&& f,
                                           Tuple&& tuple)
    // Assign left-to-right as a side-effect of initializing a temporary array
    : partial{
        { std::get<Indices + 1>(tuple) =
	    f(std::get<Indices>(tuple), std::get<Indices + 1>(tuple)) }...
	} {
  }

  discard partial[sizeof...(Indices)];
};

template <typename F, typename T, typename... Args>
constexpr bool variadic_partial_fold(F&& f, T& init, Args&... args) {
  return (void(variadic_partial_fold_sequence<std::index_sequence_for<Args...>>
    (std::forward<F>(f),
     std::forward_as_tuple(init, args...))),
    true);
}
{% endhighlight %}

Strictly speaking, the above is only `constexpr` from C++14 because of `std::get`
and `std::forward_as_tuple`, but is possible to backport this to C++11.
