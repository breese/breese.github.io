---
layout: post
title: Unifying error codes
tags: C++, error
---

C++11 and Boost both have an `error_code` class, but they cannot be used interchangeably despite their close resemblance. We introduce a trick of re-catogorizing the `error_code` from Boost to make it compatible with C++11. This enables us to use `std::error_code` throughout our entire project, and still integrate with third-party libraries that use `boost::system::error_code`.

First we have to learn how to define custom error codes.

<!--more-->

## Error codes

|![error_code](/assets/errorcode/error_code.png){: style="width:55%;"}|
|_Figure 1_: Anatomy of std::error&#95;code.|
{: class="marginimage"}

The [`<system_error>`](http://en.cppreference.com/w/cpp/header/system_error) header added in C++11 originates from [Boost.System](http://www.boost.org/libs/system/doc/index.html). The following description applies to both the C++11 and the Boost.System variants. The header contains the `error_code` class used to carry system error codes, and it can be extended with our own custom error codes as well.

When we combine error codes from different sub-systems into a single framework, each error code must be globally unique. `error_code` achieves this by containing both an error category identifying the sub-system and the sub-system-specific integral value (an enum) identifying the actual error. This way, each sub-system can use their own enum without having to worry about which values the other sub-systems have chosen. The category furthermore provide textual representations of the category and the enumerators.

|![error_category](/assets/errorcode/error_category.png){: style="width:70%;"}|
|_Figure 2_: Anatomy of std::error&#95;category.|
{: class="marginimage"}

C++11 comes with two error categories: [`std::system_category`](http://en.cppreference.com/w/cpp/error/system_category) for platform-dependent error codes, and [`std::generic_category`](http://en.cppreference.com/w/cpp/error/generic_category) for C++11-defined system error codes that originates from POSIX error codes. You can think of the latter as the former with a cross-platform enum.

We can pass an `error_code` variable around like a normal function argument or return value, or throw it via the `system_error` exception.

This construct is both extensible and reasonably efficient.

## Custom error codes

Extending `std::error_code` with custom error codes is done by:

1. Defining the custom error codes as an enum.
2. Defining a custom error category that inherits from [`std::error_category`](http://en.cppreference.com/w/cpp/error/error_category).
3. Register the custom enum within the `std::error_code` framework.

|![Custom error_code](/assets/errorcode/custom_error_code.png){: style="width:70%;"}|
|_Figure 3_: Anatomy of custom error codes.|
{: class="marginimage"}

We can then generate `std::error_code`s containing the custom error category and a custom enumerator.

The custom error codes are defined in a normal enum within a custom namespace.

{% highlight c++ %}
namespace custom
{
enum errc_t
{
    forbidden
    // Add more error values here
};
} // namespace custom
{% endhighlight %}

This custom enum must be registered using the [`std::is_error_code_enum`](http://en.cppreference.com/w/cpp/error/error_code/is_error_code_enum) trait.

{% highlight c++ %}
namespace std
{
template<>
struct is_error_code_enum<custom::errc_t>
    : public std::true_type {};
} // namespace std
{% endhighlight %}

We also want to overload `std::make_error_code()` for convenience.

{% highlight c++ %}
namespace std
{
inline std::error_code make_error_code(custom::errc_t e)
{
    return std::error_code(static_cast<int>(e),
                           custom::some_category());
}
} // namespace std
{% endhighlight %}

The custom error category must inherit from `std::error_category` and be obtainable via a singleton.

{% highlight c++ %}
namespace custom
{
const std::error_category& some_category();
} // namespace custom
{% endhighlight %}

Its implementation must override some virtual functions to provide textual representations of the error category (`name`) and enumerators (`message`.)

{% highlight c++ %}
namespace custom
{
namespace detail
{
class some_category : public std::error_category
{
public:
    virtual const char *name() const noexcept override
    {
        return "custom::category";
    }
    virtual std::string message(int value) const override
    {
        switch (errc_t(value))
        {
        case custom::forbidden:
            return "forbidden";
        // Add more descriptions here
        }
        return "unknown custom::category error";
    }
};
} // namespace detail
const std::error_category& some_category()
{
    static detail::some_category instance;
    return instance;
}
} // namespace custom
{% endhighlight %}

## Error impedance

Everything has been rosy so far, but we still have to face our original challenge of using both `std::error_code` and `boost::system::error_code` in the same application.

Let us say that we use C++11 throughout our application, but we also rely on some Boost libraries such as [Boost.Asio](http://www.boost.org/doc/html/boost_asio.html) that defines their own error codes based on Boost.System. If we want to propagate these errors, whether by value or via exceptions, we will quickly discover that we cannot pass a `boost::system::error_code` to a function that expects a `std::error_code`.

When we try to come up with workarounds, we will eventually run into the constraint that the two inherit from different `error_category` base classes. That is the root cause, but also the seed for a solution.

## Bridging the impedance

The trick to overcome the before-mentioned problem is to transform `boost::system::error_code`s into `std::error_code`s by re-categorizing the error codes.

|![Bridge error_code](/assets/errorcode/bridge_error_code.png){: style="width:100%;"}|
|_Figure 4_: Example of the boost-to-std bridge.|
{: class="marginimage"}

This is done by defining a bridge error category that acts as a placeholder for a boost error category within the `std::error_code` framework. This bridge error category inherits from `std::error_category`, so we make it look like a `std::error_category` but act like a `boost::system::error_category`.

We do not need to redefine the enum as we reuse that from Boost.System verbatim. Converting the boost error code into C++11 is then done by creating a `std::error_code` with the Boost.System enum and our bridge error category.

We start by defining our bridge error category. While Boost.System contains several error categories, we are only interested in `generic_category` for now.

{% highlight c++ %}
namespace bridge
{
const std::error_category& generic_category();
} // namespace bridge
{% endhighlight %}

The following step is a bit controversial because we register the Boost.System error values in the `std::error_code` framework.[^1] While this is legal code, it is not good style to do this on behalf of Boost.System... but here we go.

{% highlight c++ %}
namespace std
{
template<>
struct is_error_code_enum<boost::system::errc::errc_t>
    : public std::true_type
{
};
} // namespace std
{% endhighlight %}

And equally controversial, we also overload `std::make_error_code()` with the Boost.System error codes.

{% highlight c++ %}
namespace std
{
inline std::error_code make_error_code(boost::system::errc::errc_t e)
{
    return std::error_code(static_cast<int>(e),
                           bridge::generic_category());
}
} // namespace std
{% endhighlight %}

With the above we are (almost) able to create a `std::error_code` with Boost.System error values.

We still need to implement the `bridge::generic_category` singleton with overloaded functions that returns the names of the error category and the error values. Rather than providing the textual representations of all error values ourselves we leverage the fact that `boost::system::generic_category` already is able to do this.[^2]

{% highlight c++ %}
namespace bridge
{
namespace detail
{
class generic_category : public std::error_category
{
public:
    virtual const char *name() const noexcept override
    {
        return boost::system::generic_category().name();
    }
    virtual std::string message(int value) const override
    {
        return boost::system::generic_category().message(value);
    }
};
} // namespace detail
const std::error_category& generic_category()
{
    static detail::generic_category instance;
    return instance;
}
} // namespace bridge
{% endhighlight %}

The above procedure can also be applied to Boost.Asio error codes.

## Conversion

All that is left now is to do the actual conversion.

{% highlight c++ %}
namespace bridge
{
inline std::error_code convert(const boost::system::error_code& error)
{
    if (error.category() == boost::system::generic_category())
    {
        return std::error_code(error.value(),
                               bridge::generic_category());
    }
    assert(false);
    return {}; // We could define a bridge conversion error instead
}
} // namespace bridge
{% endhighlight %}

Disclaimer: I ought to have used `error_condition` instead of `error_code` for custom error codes in the above, but as the distinction between platform-dependent and platform-independent error codes can be expressed via error categories, I just do not see the point of `error_condition`.

[^1]: A less controversial solution is to import the Boost.System enum into our custom namespace, and then register that instead.

[^2]: This is not only the easiest, but also the most correct solution.