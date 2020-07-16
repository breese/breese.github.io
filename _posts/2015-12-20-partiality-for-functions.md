---
layout: post
title: Partiality for Functions
tags: C++, template
---

While C++ does support partial specialization of templates, it does not do so for function templates. Instead, the general advice is to use function overloading instead. Sometimes that is not a feasible solution though, so we will see how to emulate partial specialization of function template with a some boiler-plate code.

<!--more-->

## Partial specialization of function templates

Assume that we have a `formatter` class that converts C++ values of different types into text, such as a JSON writer. We want this class to have a consistent API where any kind of value is written with the `formatter::write` function.

{% highlight c++ %}
// Create the formatter
formatter output;

// Write an integer
output.write(1);

// Write a map
std::map<std::string, int> dictionary = { { "alpha", 1 } };
output.write(dictionary);
{% endhighlight %}

The obvious solution to handle any kind of type is to use a member function template for `formatter::write`.

{% highlight c++ %}
class formatter
{
public:
    template <typename T>
    void write(const T&);
};
{% endhighlight %}

As different types have to be formatted in different ways, we need specializations for the different types that our class is going to support. The `int` case from the example above is easily added using function overloading.

{% highlight c++ %}
class formatter
{
public:
    template <typename T>
    void write(const T&);

    // Added overload for int
    void write(int);
};
{% endhighlight %}

Adding support for `std::map` is more tricky, because we want to invoke different formatting depending on the `key_type` of the map. For instance, when `key_type` is a string then we want a JSON formatter to output the map as a JSON object, but for all other data types we want to output it as a JSON array of pairs. This is not an arbitrary restriction that I have dreamt up; this is how JSON is defined.

Ideally we would like to be able to write the following:

{% highlight c++ %}
class formatter
{
public:
    template <typename T>
    void write(const T&);

    // Added general map case (illegal)
    template <typename Key, typename Value>
    void write(const std::map<Key, Value>&);

    // Added special map case (illegal)
    template <typename Value>
    void write(const std::map<std::string, Value>&);

    void write(int);
};
{% endhighlight %}

Unfortunately the above specializations of the `write` function are partial and therefore not legal in C++. We need something else to resolve the different map cases. Function overloading cannot be used in this case either, because all types must be fully specialized but our `Value` parameter is not specialized in either case.

## Indirect specialization

As in so many other situations, we are going to overcome the limitation with another level of indirection. The basic idea is this:

> Use partial specialization of templates to emulate partial specialization of function templates.

In our second attempt, our `formatter::write` is a single template function that uses forwarding references, and consequently all function overloads have been removed.[^1] This function forwards any call to a helper class called `formatter::overloader`, which will handle partial specialization for us.

{% highlight c++ %}
class formatter
{
public:
    template <typename T>
    void write(T&& value)
    {
        // Bounce to helper class
        using type = typename std::decay<T>::type;
        overloader<type>::write(*this, std::forward<T>(value));
    }

private:
    template <typename T, typename Enable = void>
    struct overloader;
};
{% endhighlight %}

The actual formatting implementations are added as the uniquely named private member functions `write_integral`, `write_map`, and `write_string_map` in the `formatter` class.[^2]

{% highlight c++ %}
class formatter
{
public:
    template <typename T>
    void write(T&& value)
    {
        // Bounce to helper class
        using type = typename std::decay<T>::type;
        overloader<type>::write(*this, std::forward<T>(value));
    }

private:
    template <typename T, typename Enable = void>
    struct overloader;

    // Added int case
    template <typename T>
    void write_integral(const T&);

    // Added general map case
    template <typename T>
    void write_map(const T&);

    // Added special map case
    template <typename T>
    void write_string_map(const T&);
};
{% endhighlight %}

The `formatter::write` function forwards calls to the `formatter::overloader<T>::write` function. We first need to define the general case. This should fail if our input type does not match any of our overloads.

{% highlight c++ %}
// Matches all non-specialized types
template <typename T, typename Enable>
struct formatter::overloader
{
    // No implementation for the general case
};
{% endhighlight %}

Next we define the `int` case. Let us extend this case to handle any integral type while we are at it. The `write` function simply calls the appropriate private implementation function on the `formatter` class.

{% highlight c++ %}
// Matches any integral type
template <typename T>
struct formatter::overloader<
    T,
    typename std::enable_if<std::is_integral<T>::value>::type
    >
{
    inline static
    void write(formatter& self, const T& value)
    {
        // Bounce back into the formatter class
        self.write_integral(value);
    }
};
{% endhighlight %}

Finally we add the two different `std::map` cases.

{% highlight c++ %}
// Matches maps with non-string keys
template <typename Key, typename Value>
struct formatter::overloader<
    std::map<Key, Value>
    >
{
    using type = std::map<Key, Value>;

    inline static
    void write(formatter& self, const type& value)
    {
        self.write_map(value);
    }
};

// Matches maps with string keys
template <typename Value>
struct formatter::overloader<
    std::map<std::string, Value>
    >
{
    using type = std::map<std::string, Value>;

    inline static
    void write(formatter& self, const type& value)
    {
        self.write_string_map(value);
    }
};
{% endhighlight %}

And that is that. A lot of boiler-plate is needed, but is doable to emulate partial specialization of function templates in C++. The examples above used C++11 for convenience, but this technique can also be written in C++03 with the use of Boost type-traits.

[^1]: Read _Item 26: Avoid overloading on universal references_ in Scott Meyers ``Effective Modern C++'' if you wonder why.

[^2]: These implementation functions are part of the boiler-plate code, not of the API. We could just as well have placed them in the `formatter::overloader` class. That is simply an implementation detail.
