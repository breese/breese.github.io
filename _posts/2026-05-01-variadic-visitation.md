---
layout: post
title: Variadic Visitation
tags: C++. Template
---

Variadic visitation selects an element from a parameter pack using a run-time index,
unlike parameter pack indexing that uses a compile-time index.

<!--more-->

C++26 [parameter pack indexing](https://en.cppreference.com/cpp/language/pack_indexing)
only works with compile-time indices, and for a good reason. Consider a function
that returns the Nth element of a parameter pack

{% highlight c++ %}
template <size_t N>
auto pack_at(integral_constant<size_t, N> index, auto... args)
{
  return args...[N];
}
{% endhighlight %}

The parameter pack may contain different types, and as C++ is statically typed
the return type must be deduced at compile-time. Therefore, the index must be known
at compile-time.

Our goal is to use a run-time index instead

{% highlight c++ %}
auto variadic_at(size_t index, auto... args)
{
  return args...[index]; // Fails... for now
}
{% endhighlight %}

The code below assumes C++17 to use standard library facilities, such as
`std::variant` or `std::apply`, but if you have alternative implementations
of these, then the code can be back-ported to C++14, and even C++11 by
adding the appropriate trailing return types.

## Visitation

The `variadic_at` function could be changed to return `std::variant`

{% highlight c++ %}
auto variadic_at(size_t index, auto... args) -> variant<decltype(args)...>;
{% endhighlight %}

The value inside the variant is then accessed using `std::visit`

{% highlight c++ %}
auto r = variadic_at(index, args...);
std::visit([] (auto&& v) { cout << v; }, r);
{% endhighlight %}

Since we end up using a visitor anyways, we might as well start by
building a variadic visitor solution.

Getting variadic visitation to work involves a number of tricks, and
this is the most complicated part of this article so the implementation
is deferred until the end.
Once we have this building-block in place, everything else becomes simple.

The variadic visitation is declared as

{% highlight c++ %}
template <typename Visitor, typename... Args>
constexpr auto variadic_visit(Visitor&&, size_t index, Args&&...);
{% endhighlight %}

and used like this

{% highlight c++ %}
variadic_visit(f, 0, a, b, c); // Invokes f(a)
variadic_visit(f, 1, a, b, c); // Invokes f(b)
variadic_visit(f, 2, a, b, c); // Invokes f(c)
variadic_visit(f, 3, a, b, c); // Error: out-of-bounds
{% endhighlight %}

This example uses literal indices to demonstrate how variadic visitation works,
but it can be used with index variables as well.

## Tuple Visitation

Tuple visitation can be build on top of variadic visitation.
The trick is to use `std::apply` to expand the tuple elements into a parameter pack.

The `variadic_visit` function takes a visitor, an index, and a parameter pack,
but `std::apply` only supplies the parameter pack so we need to capture the visitor
and index.
We will use a lambda function that captures the visitor as a tuple to preserve
perfect forwarding.

{% highlight c++ %}
template <typename Visitor, type-like Tuple>
constexpr auto tuple_visit(Visitor&& v, size_t index, Tuple&& t)
{
  return apply([v = forward_as_tuple(v), index] (auto&&... args) {
                 return variadic_visit(forward<Visitor>(get<0>(v)),
		                       index,
				       forward<decltype(args)>(args)...);
               },
	       forward<Tuple>(t));
}
{% endhighlight %}

Usage

{% highlight c++ %}
auto t = make_tuple(a, b, c);
tuple_visit(f, 0, t); // Invokes f(a)
tuple_visit(f, 1, t); // Invokes f(b)
tuple_visit(f, 2, t); // Invokes f(c)
tuple_visit(f, 3, t); // Error: out-of-bounds
{% endhighlight %}

## Returning variant

Now we are able to create a function that returns a variant.
The idea is to use a visitor that creates objects, and for that we use
a slightly altered implementation of the [P3841](https://wg21.link/P3841) proposal.

{% highlight c++ %}
template <typename T>
struct constructor
{
  template <typename... Args>
  constexpr T operator()(Args&&... args) const
    noexcept(is_nothrow_constructible_v<T, Args...>)
  {
    return T(forward<Args>(args)...);
  };
};
{% endhighlight %}

The variant type is deduced from the argument types.

{% highlight c++ %}
template <typename... Args>
constexpr auto variadic_at(size_t index, Args&&... args)
{
  using result_type = variant<Args...>;
  return variadic_visit(constructor<result_type>{},
                        index,
			forward<Args>(args)...);
}
{% endhighlight %}

It would be a good idea to remove duplicated types from the variant, but we
have skipped over that for brevity.

The same can be done for tuples, where the tuple-like type must be transformed
into a variant type.

{% highlight c++ %}
template <tuple-like Tuple>
constexpr auto tuple_at(size_t index, Tuple&& t)
{
  using result_type = deduce_variant_t<Tuple>;
  return tuple_visit(constructor<result_type>{},
                     index,
                     forward<Tuple>(t));
}
{% endhighlight %}

where the `deduce_variant_t` must account for all tuple-like types, including
arrays.

{% highlight c++ %}
template <typename, typename>
struct deduce_variant;

template <typename T, size_t... Index>
struct deduce_variant<T, index_sequence<Index...>>
{
  using type = std::variant<tuple_element_t<Index, T>...>;
};

template <typename T>
using deduce_variant_t
  = typename deduce_variant<T, make_index_sequence<tuple_size_v<T>>>::type;
{% endhighlight %}

Again, it would be a good idea to remove duplicated types.

## Variadic Visitation

Variadic visitation works in a similar way to `std::visit`.
It builds a look-up table to map the run-time index into a typed function
whose signature contains the compile-time index.
An exception is thrown if the run-time index is out-of-bounds.

{% highlight c++ %}
template <typename Visitor,
         typename... Args,
	 R = common_type_t<invoke_result_t<Visitor>, Args>...>>
constexpr R variadic_visit(Visitor&& v, size_t index, Args&&... args)
{
  using table_type = variadic_visit_table<R,
                                          index_sequence_for<Args...>,
                                          Visitor,
                                          Args...>;
  // Immediately invoked function object
  return (index < sizeof...(Args))
    ? table_type{}(forward<Visitor>(v), index, forward<Args>(args)...)
    : throw out_of_range{"variadic_visit"};
}
{% endhighlight %}

The return type `R` of variadic visitation is deduced by considering
the return types of all the potential visitor invocations.

The `variadic_visit_table` helper contains the look-up table and does
all the necessary type erasure.
The injected integer sequence is needed to initialize array.

{% highlight c++ %}
template <typename, typename, typename, typename...>
struct variadic_visit_table;

template <typename R, size_t... Index, typename Visitor, typename... Args>
struct variadic_visit_table<R, index_sequence<Index...>, Visitor, Args...>
{
private:
  template <size_t N>
  static constexpr R indexed_invoke(Visitor&& v, Args&&... args)
  {
    // Invokes visitor with Nth element
    return invoke_r<R>(forward<Visitor>(v),
                       pack_at(integral_constant<size_t, N>{},
		               forward<Args>(args)...));
  }

  // Array of function pointers
  decltype(&indexed_invoke<0>) table[sizeof...(Index)];

public:
  constexpr variadic_visit_table()
    : table{ &indexed_invoke<Index>... } // Parameter pack expansion
  {
  }

  // Preconditions:
  // - index is within bounds.
  constexpr R operator()(Visitor&& v, size_t index, Args&&... args) const
  {
      return table[index](forward<Visitor>(v), forward<Args>(args)...);
  }
};
{% endhighlight %}

The `pack_at` function is similar to that defined earlier, but needs to
be back-ported from C++26 to earlier standards.

{% highlight c++ %}
template <size_t N, typename... Args>
auto pack_at(integral_constant<size_t, N> index, Args&&... args)
{
#if __cpp_pack_indexing >= 202311L
  return args...[N];
#else
  return get<N>(forward_as_tuple(forward<Args>(args)...));
#endif
}
{% endhighlight %}

Caveat: The above has glossed over some technical detail, such as using `decltype(auto)`
rather than `auto` as the return type.
