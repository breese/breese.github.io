---
layout: post
title: Customizing Error Codes
tags: C++, error
---

Many C libraries follow the [POSIX tradition](http://pubs.opengroup.org/onlinepubs/9699919799/basedefs/errno.h.html) of returning errors as integers. Some libraries reuse the pre-defined [`errno`](http://en.cppreference.com/w/cpp/header/cerrno) values, while others define their own library-specific values.

We show how to integrate the error values of such C libraries with `std::error_code`. The choice of `std::error_code` gives us several advantages: (i) errors are type-erased, (ii) errors are propagated without loss of information, and (iii) errors are interoperable.

<!--more-->

## Type-erased error

|![error_code](/assets/customize/error_code.png){: style="width:55%;"}|
|_Figure 1_: Anatomy of std::error&#95;code.|
{: class="marginimage"}

[`std::error_code`](http://en.cppreference.com/w/cpp/error/error_code) holds two member variables: an error value of type `int`, and a pointer to a library-specific category that represents the error type. The category is a singleton which inherits from [`std::error_category`](http://en.cppreference.com/w/cpp/error/error_category).

Each library must define its own error category that overrides the `name()` and `message()` functions of `std::error_category`. The former returns the name of the category, and the latter a textual representation of a concrete error value. The category can be considered a polymorphic [`std::type_info`](http://en.cppreference.com/w/cpp/types/type_info) that must be implemented manually and therefore works without [RTTI](https://en.wikipedia.org/wiki/Run-time_type_information).

Two `std::error_code` objects are equal if they have the same error value and they point to the same category singleton.

`std::error_code` is type-erased, which means that its type remains the same regardless of which error types it embeds. This is achieved because the error category is a polymorphic singleton, and because `std::error_code` stores the address of a category singleton, no [object-slicing](https://en.wikipedia.org/wiki/Object_slicing) will take place when `std::error_code` is copied. Polymorphism is used here as a type-erasure technique.

Type-erasure gives us some advantages:

0. Type-erasure enables us to build [compilation firewalls](http://en.cppreference.com/w/cpp/language/pimpl), whereby our C++ headers do not accidentially expose macros and global symbols from the underlying C library.
0. Lossless propagation means that although the error is type-erased, we preserve both the original error value from the C library and its type. The type is not preserved directly but can be deduced from the category. If an upgrade of the C library introduces new error values, they will be propagated even if we do not upgrade our C++ adapters.
0. Errors from different categories can be compared with each other because the error category has a concept for equivalence. Equivalence makes it easier to add new C libraries to the fray as demonstrated below.

## Audio codecs as example

The running example here is C++ adaptors for [audio codec](https://en.wikipedia.org/wiki/Audio_codec) libraries.

These libraries are particularly interesting for our purposes, because although they implement different audio codecs, and they are written by different developers, their error values share a [family-resemblance](https://en.wikipedia.org/wiki/Family_resemblance) -- there is sufficient overlap between their error values so they can be handled in the same manner. For instance, all audio codecs support a limited amount of sample rates, and they return an error if the caller selects an unsupported sample rate.

At the same time, there are also differences so we cannot translate the error values into a pre-defined set of common errors. We therefore want to propagate the actual error value without loss of information, but we only want to compare and react to well-known errors.

`std::error_code` was originally designed to allow multiple subsystems to encapsulate their own error values in a common error container. We can therefore use `std::error_code` to propagate the error values from all audio codec libraries. We just have to define an error category for each library.

Furthermore, the audio decoders are paired with a transport mechanism that feeds them with data -- e.g. a file reader or network streaming -- so we must be able to pass through transport errors as well. We are going to assume that errors from the transport layer are already wrapped in `std::error_code`. If not, the techniques described below can be used.

## Native adapters

|![mpg123 adapter](/assets/customize/mpg123_error.png){: style="width:70%;"}|
|_Figure 2_: Native adapter for mpg123.|
{: class="marginimage"}

Suppose we want to create a C++ adapter for the [mpg123](https://mpg123.org/) audio codec library. Below is a selective part of its error values.

{% highlight c %}
// mpg123.h

enum mpg123_errors
{
  MPG123_OK = 0,
  MPG123_BAD_RATE = 3,
  MPG123_BAD_PARAM = 5,
  MPG123_OUT_OF_MEMORY = 7,
  // ...
};

const char *mpg123_plain_strerror(int);
{% endhighlight %}

The `mpg123_plain_strerror()` function converts an error value into a textual representation.

The C++ adapter simply consists of an error category and a `make_error_code()` function that is used to convert mpg123 error values into `std::error_code`.

{% highlight c++ %}
// mpg123.hpp

#include <system_error>

namespace mpg123
{

const std::error_category& category();

inline std::error_code make_error_code(int mpg123_value)
{
  // Create an error_code with the original mpg123 error value
  // and the mpg123 error category.
  return std::error_code(mpg123_value, mpg123::category());
}

} // namespace mpg123
{% endhighlight %}

Notice that this header does not include the C library header `<mpg123.h>`, so we have not polluted the global namespace with symbols from the C library.

Most of the implementation is boiler-plate code, whose main purpose is to delegate the category `message()` call to the `mpg123_plain_strerror()` function.

{% highlight c++ %}
// mpg123.cpp

#include <mpg123.h> // C header
#include <mpg123.hpp> // Adapter

namespace mpg123
{
namespace detail
{

class category : public std::error_category
{
public:
  virtual const char *name() const noexcept override
  {
    return "mpg123::category";
  }
  virtual std::string message(int value) const override
  {
    // Let the native function do the actual work
    return ::mpg123_plain_strerror(value);
  }
};

} // namespace detail

const std::error_category& category()
{
  // The category singleton
  static detail::category instance;
  return instance;
}

} // namespace mpg123
{% endhighlight %}

We can define adapters for other audio codec libraries in a similar manner.

### Using the adapter

Whenever the C++ adapter encounters an error from the mpg123 library, we can use `mpg123::make_error_code()` to return a `std::error_code` to the caller.

{% highlight c++ %}
  // ...
  int error = mpg123_getformat(handle, &rate, &channels, &encoding);
  if (error != MPG123_OK)
    return mpg123::make_error_code(error);
{% endhighlight %}

The caller then has several options:

0. Print the error. This is done with `error_code::message()`.
0. Throw the error. Wrap the `error_code` in a [`std::system_error`](http://en.cppreference.com/w/cpp/error/system_error) exception. Better yet, define an `mpg123::error` exception that inherits from `std::system_error`, so exceptions from the mpg123 adapter can be distinguished from other `system_error` exceptions in the try-catch block.
0. Check the error value. As the `error_code` contains the original error value, the caller must first check that the error code belongs to the `mpg123::category` and then compare the `error_code::value()` against `mpg123_errors` enumerators defined in `<mpg123.h>`.

If the comparison is done in a `.cpp` file, then all options above can be done behind a compilation firewall.

{% highlight c++ %}
// caller.cpp

#include <mpg123.h>
#include <mpg123.hpp>

void inform(const std::error_code& error)
{
  if (error.category() == mpg123::category())
  {
    switch (error.value())
    {
    case MPG123_OK:
      break; // Nothing to report
    case MPG123_BAD_RATE:
      inform_bad_format();
      break;
    case MPG123_BAD_PARAM:
      inform_illegal_argument();
      break;
    case MPG123_OUT_OF_MEMORY:
      std::terminate();
    }
  }
}
{% endhighlight %}

This is a manageable solution, albeit not an elegant one. When we add another audio codec, say [libopus](https://opus-codec.org/), then the caller must check for its error values as well.

{% highlight c++ %}
// caller.cpp

#include <mpg123.h>
#include <opus/opus.h>
#include <mpg123.hpp>
#include <opus.hpp>

void inform(const std::error_code& error)
{
  if (error.category() == mpg123::category())
  {
    switch (error.value())
    {
    case MPG123_OK:
      break; // Nothing to report
    case MPG123_BAD_RATE:
      inform_bad_format();
      break;
    case MPG123_BAD_PARAM:
      inform_illegal_argument();
      break;
    case MPG123_OUT_OF_MEMORY:
      std::terminate();
    }
  }
  else if (error.category() == opus::category())
  {
    switch (error.value())
    {
    case OPUS_OK:
      break; // Nothing to report
    case OPUS_BAD_ARG:
      inform_illegal_argument();
      break;
    case OPUS_INVALID_PACKET:
      inform_bad_format();
    case OPUS_ALLOC_FAIL:
      std::terminate();
    }
  }
}
{% endhighlight %}

Although our printing and throwing use cases are really simple, the checking use case places a heavy burden on the user of our C++ adapters, and the burden grows for each audio codec we wrap. We need something better.

## Generic enumeration

|![Generic adapter](/assets/customize/codec_error.png){: style="width:70%;"}|
|_Figure 3_: Generic adapter with enumerators.|
{: class="marginimage"}

A more user-friendly way of checking errors is to define enumerators to compare with. We notice that some of the native error values are already covered by [`std::errc`](http://en.cppreference.com/w/cpp/error/errc), such as `invalid_argument`, so we will not include them in our own enumeration.

The caller should be able to write the following. Notice the omission of the C library `<mpg123.h>` and `<opus/opus.h>` headers. This means that we maintain the compilation firewall.

{% highlight c++ %}
// caller2.cpp

#include <codec.hpp>

void inform(const std::error_code& error)
{
  if (error == codec::illegal_sample_rate)
    inform_bad_format();
  else if (error == std::errc::invalid_argument)
    inform_invalid_argument();
  else if (error == std::errc::not_enough_memory)
    std::terminate();
}
{% endhighlight %}

We define a generic enumeration and error category that we want to use across all audio codec libraries.

{% highlight c++ %}
// codec.hpp

#include <system_error>

namespace codec
{

enum class errc
{
  success = 0,
  illegal_sample_rate,
  illegal_sample_width
};

const std::error_category& category();
std::error_code make_error_code(codec::errc);
std::error_condition make_error_condition(codec::errc);

} // namespace codec
{% endhighlight %}

As the native error values and the `codec::errc` enumeration have different types and numerical values, we need a mapping between them. This mapping is done by our category's `equivalent()` member function.

{% highlight c++ %}
// mpg123.cpp

#include <mpg123.h>
#include <codec.hpp>
#include <mpg123.hpp>

namespace mpg123
{
namespace detail
{

class category : public std::error_category
{
public:

  // ...

  // Compare own value with foreign condition
  virtual bool equivalent(
    int mpg123_value,
    const std::error_condition& condition) const noexcept override
  {
      switch (mpg123_value)
      {
      case MPG123_OK:
          return bool(condition);
      case MPG123_BAD_RATE:
          return condition ==
            codec::make_error_condition(codec::errc::illegal_sample_rate);
      case MPG123_BAD_PARAM:
          return condition ==
            std::make_error_condition(std::errc::illegal_argument);
      case MPG123_OUT_OF_MEMORY:
          return condition ==
            std::make_error_condition(std::errc::not_enough_memory);
      default:
          return false;
      }
  }
};

} // namespace detail

// ...

} // namespace mpg123
{% endhighlight %}

Finally we have to create a specialization of [`std::is_error_condition_enum`](http://en.cppreference.com/w/cpp/error/error_condition/is_error_condition_enum) so our enum can be compared directly with a `std::error_code`.

{% highlight c++ %}
// codec.hpp

namespace std
{

template <>
struct is_error_condition_enum<codec::errc>
  : public std::true_type {};

} // namespace std
{% endhighlight %}

The advantage of the above is that we have moved all knowledge of the mpg123 library back into the C++ adapter.

## Error condition

We have used [`std::error_condition`](http://en.cppreference.com/w/cpp/error/error_condition) in the comparisons in the C++ adapter. We cannot use `std::error_code` because it will perform an exact match. Instead `std::error_condition` was introduced to enable equivalent comparisons.

The descriptions of `std::error_condition` are a bit vague. The C++ standard $[$syserr.errcondition.overview$]$ states that

> The class error_condition describes an object used to hold values identifying error conditions. $[$ Note: error_condition values are portable abstractions, while error_code values (19.5.3) are implementation specific. — end note $]$

with no elaboration on what an error condition is. This leaves us with an impression that `std::error_condition` could be used to propagate platform-independent errors. This is collaborated by the fact that its interface is almost identical to `std::error_code`.

That is not the case. As one of the original designers [explains](http://blog.think-async.com/2010/04/system-error-support-in-c0x-part-1.html)

> class error_condition - something that you want to test for and, potentially, react to in your code.

Therefore, a more useful guideline is to:

* Use `std::error_code` for error propagation.
* Use `std::error_code` for comparison within a category.
* Use `std::error_condition` for comparison between categories.
