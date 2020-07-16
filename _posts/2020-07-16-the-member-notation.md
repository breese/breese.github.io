---
layout: post
title: The Member Notation
tags: C++
---

Coding guidelines often introduce a special notation to distinguish member variables from function parameters. This notation is not known to the compiler, so it cannot detect violations. At best we can write rules for static analysis tools, such as clang-tidy, to detect violations.

But there is an alternative supported by the compiler, namely nested classes.

<!--more-->

## A Class Within a Class

<p class="margintext">
<em>A class can be declared within another class. A class declared within another is called a nested class. The name of a nested class is local to its enclosing class. The nested class is in the scope of its enclosing class.</em>
<br/>
-- C++ Standard N4849, section $[$class.nest$]$/1
<br/><br/>
<em>A nested class is a member and as such has the same access rights as any other member.</em>
<br/>
C++ Standard N4849, section $[$class.access.nest$]$/1
</p>

A [nested class](https://en.cppreference.com/w/cpp/language/nested_types) is simply a class that is declared inside another class.

A nested class is a member of the outer class, so it has full access rights to the members of the outer class. The obverse is not true -- the outer class only has public access right to the members of the nested class.

A nested class is not an inner class. The nested object does not know which enclosing object it belongs to, unless the `this` pointer of the enclosing object is passed to the nested object.

## Into Shadows

Let us revisit the problem we are trying to address.

<p class="margintext">
<em>A declaration of a name in a nested declarative region hides a declaration of the same name in an enclosing declarative region.</em>
<br/>
C++ Standard N4849, section $[$basic.scope.hiding$]$/1
</p>

C++ allows [variable shadowing](https://en.wikipedia.org/wiki/Variable_shadowing) where we can use a variable name in an inner scope even though another variable with the same name already exists in an outer scope.

Shadowing, however, can lead to undetected defects in member functions where parameters may shadow member variables.

{% highlight c++ %}
struct shadowing
{
  // Error: Assigns parameter to itself; member variable is untouched.
  void set(int value) { value = value; }
 
private:
  int value;
};
{% endhighlight %}

The problem is even more subtle for constructors because there is no shadowing in [member initializer lists](http://en.cppreference.com/w/cpp/language/initializer_list). The compiler can correctly choose between constructor parameters and member variables with the same name, but once you get into the body of the constructor the usual shadowing applies again.

{% highlight c++ %}
struct body_shadowing
{
  body_shadowing(int value)
    : value(value) // Ok: Assigns parameter to member variable.
  {
    // Error: Assigns parameter to itself; member variable is untouched.
    value = value;
  }
 
private:
  int value;
};
{% endhighlight %}

Usual workarounds are to rely on compiler warnings, const parameters, the `this` pointer, or naming conventions to address this problem, but each have their own set of limitations.

## Nested Member Variables

Instead, we are going to create a dedicated "namespace" for member variables by putting them into a nested class.

{% highlight c++ %}
struct class_with_nested_member
{
  // Ok: Assigns parameter to member variable using aggregate initialization.
  class_with_nested_member(int value) : member{value} {}
 
  // Ok: Assigns parameter to member variable.
  void set(int value) { member.value = value; }
 
private:
  // Anonymous nested class containing the member variables.
  struct
  {
    int value;
  } member;
};
{% endhighlight %}

Our member variable is now accessed with `member.value`.

Unlike naming conventions such as a prefixed `m_` or a suffix underscore, where we accidentally could name a function parameter `m_value` or `value_`, we cannot accidentally name the function parameter `member.value`. In other words, we have effectively created a naming convention for member variables that can be verified by the compiler.

We can pick any legal variable name for our nested class, so we could have used `m.value` if we wanted to have a syntax close to the `m_value` convention. However, we should prefer expressive names over abbreviations because it reduces the cognitive load on new project members.

There is no run-time overhead of using a nested member class. The compiler will generate the same code as an equivalent class without  a nested class, even when optimization is turned off.

## Aggregate Initialization

<p class="margintext">
<em>An aggregate is an array or a class with<br/>
    – no user-declared or inherited constructors,<br/>
    – no private or protected direct non-static data members,<br/>
    – no virtual functions,<br/>
    – no virtual, private, or protected base classes.</em>
<br/>
C++ Standard N4849, section $[$dcl.init.aggr$]$/1
</p>

In the previous example we used [aggregate initialization](https://en.cppreference.com/w/cpp/language/aggregate_initialization) of the `member` variable, which saves us from writing a constructor for the nested class. Although that works fine in many cases, there are situations were we cannot rely on aggregate initialization.

When aggregate initialization is not possible, we need to add a constructor to the nested `member` class. Narrowing also prevents aggregate initialization.

In the remainder we are going to look at some special cases.

## Non-Copyable Members

Let us say we want to use an std::atomic<T> member variable. We cannot initialize an atomic variable with an assignment operator because that would involve copying.

{% highlight c++ %}
// Error: std::atomic is not copyable.
std::atomic<int> x = 42;
 
// Ok: Braced initialization with 42.
std::atomic<int> x{42};
{% endhighlight %}

If we have an atomic member variable, then we need to use the same kind of initialization.

{% highlight c++ %}
struct with_copy_construction
{
  // Error: Attempts to copy construct the atomic variable
  with_copy_construction(int value) : member{value} {}
 
private:
  struct
  {
    std::atomic<int> value;
  } member;
};
{% endhighlight %}

{% highlight c++ %}
struct with_braced_initialization
{
  // Ok: Braced initialization of atomic variable
  with_braced_initialization(int value) : member{ {value} } {}
 
private:
  struct
  {
    std::atomic<int> value;
  } member;
};
{% endhighlight %}

Notice the extra set of brackets in the aggregate initialization above.

## Static Member Variables

<p class="margintext">
<em>A static data member shall not be a direct member of an unnamed $[$...$]$ class</em>
<br/>
C++ Standard N4849, section $[$class.static.data$]$/2
</p>

We cannot add static members to an anonymous nested class.

This means that we need to pull all our static member variables out into the enclosing class.

{% highlight c++ %}
struct static_in_nested
{
  static_in_nested(int value) : member{value} {}
 
private:
  struct
  {
    // Error: Static data not allowed in nested classes.
    static constexpr int constant = 42;
    int value;
  } member;
};
{% endhighlight %}

{% highlight c++ %}
struct static_in_enclosing
{
  static_in_enclosing(int value) : member{value} {}
 
private:
  // Ok
  static constexpr int constant = 42;
  struct
  {
    int value;
  } member;
};
{% endhighlight %}

## Pimpl Idiom

If we want to hide the implementation details of our member variables, then the [pimpl idiom](http://en.cppreference.com/w/cpp/language/pimpl) is a natural extension of the nested class notation.

{% highlight c++ %}
struct class_with_pimpl
{
  class_with_pimpl(int value) : member(std::make_unique<member_storage>(value)) {}
 
private:
  struct member_storage; // Forward declaration of nested type
  std::unique_ptr<member_storage> member;
};
{% endhighlight %}

Now the member variables have to be accessed as `member->value`, but otherwise the notation remains the same.

## Compile-time Selection of Member Variables

Sometimes template classes need to select member variables at compile-time. This can be done using a template nested class with specializations.

Consider [`std::span`](https://en.cppreference.com/w/cpp/container/span). When using a dynamic extent, the span size is determined at construction time and thus needs to be stored as a member variable. When using a fixed extent, the span size is encoded into the type, and the `size` member variable is unnecessary.
A simplified span could look as follows, ignoring empty base optimization as the real span needs other member variables.

{% highlight c++ %}
template <typename T, std::size_t Extent = std::dynamic_extent>
class span
{
  template <typename Iterator>
  span(Iterator first, size_type count)
 
  size_type size() const { return member.size(); }
 
private:
  // Members for fixed extent
  template <typename, size_type E>
  struct member_storage
  {
    member_storage(size_type count) { assert(count == size()); }
 
    size_type size() const
    {
      return E;
    }
  };
 
  // Members for dynamic extent
  template <typename T1>
  struct member_storage<T1, std::dynamic_extent>
  {
    member_storage(size_type count) : capacity(count) {}
 
    size_type size() const
    {
        return capacity;
    }
 
    size_type capacity;
  };
 
  struct member_storage<T, Extent> member;
};
{% endhighlight %}

Notice that we maintain the `member.` notation, although we need to use a nested member function to access member variables that could be omitted.
