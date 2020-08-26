---
title: '[wiki]Substitution failure is not an error'
date: 2020-08-26 07:27:35
tags: Cpp
categories:
cover: https://miro.medium.com/max/2730/1*tgkr8r27dVIQtsEygITANA.png
---
<meta name="referrer" content="no-referrer" />

From Wikipedia, the free encyclopedia

**Substitution failure is not an error** (**SFINAE**) refers to a situation in [C++](https://en.wikipedia.org/wiki/C%2B%2B) where an invalid substitution of [template](https://en.wikipedia.org/wiki/Template_(programming)) parameters is not in itself an error. David Vandevoorde first introduced the acronym SFINAE to describe related programming techniques.[[1\]](https://en.wikipedia.org/wiki/Substitution_failure_is_not_an_error#cite_note-1)

Specifically, when creating a candidate set for [overload resolution](https://en.wikipedia.org/wiki/Overload_resolution), some (or all) candidates of that set may be the result of instantiated templates with (potentially deduced) template arguments substituted for the corresponding template parameters. If an error occurs during the substitution of a set of arguments for any given template, the compiler removes the potential overload from the candidate set instead of stopping with a compilation error, provided the substitution error is one the C++ standard grants such treatment.[[2\]](https://en.wikipedia.org/wiki/Substitution_failure_is_not_an_error#cite_note-2) If one or more candidates remain and overload resolution succeeds, the invocation is well-formed.

## Example

The following example illustrates a basic instance of SFINAE:

```cpp
struct Test {
  typedef int foo;
};

template <typename T>
void f(typename T::foo) {}  // Definition #1

template <typename T>
void f(T) {}  // Definition #2

int main() {
  f<Test>(10);  // Call #1.
  f<int>(10);   // Call #2. Without error (even though there is no int::foo)
                // thanks to SFINAE.
}
```

Here, attempting to use a non-class type in a qualified name (`T::foo`) results in a deduction failure for `f<int>` because `int` has no nested type named `foo`, but the program is well-formed because a valid function remains in the set of candidate functions.

Although SFINAE was initially introduced to avoid creating ill-formed programs when unrelated template declarations were visible (e.g., through the inclusion of a header file), many developers later found the behavior useful for compile-time introspection. Specifically, it allows a template to determine certain properties of its template arguments at instantiation time.

For example, SFINAE can be used to determine if a type contains a certain typedef:

```cpp
#include <iostream>

template <typename T>
struct has_typedef_foobar {
  // Types "yes" and "no" are guaranteed to have different sizes,
  // specifically sizeof(yes) == 1 and sizeof(no) == 2.
  typedef char yes[1];
  typedef char no[2];

  template <typename C>
  static yes& test(typename C::foobar*);

  template <typename>
  static no& test(...);

  // If the "sizeof" of the result of calling test<T>(nullptr) is equal to
  // sizeof(yes), the first overload worked and T has a nested type named
  // foobar.
  static const bool value = sizeof(test<T>(nullptr)) == sizeof(yes);
};

struct foo {
  typedef float foobar;
};

int main() {
  std::cout << std::boolalpha;
  std::cout << has_typedef_foobar<int>::value << std::endl;  // Prints false
  std::cout << has_typedef_foobar<foo>::value << std::endl;  // Prints true
}
```

When `T` has the nested type `foobar` defined, the instantiation of the first `test` works and the null pointer constant is successfully passed. (And the resulting type of the expression is `yes`.) If it does not work, the only available function is the second `test`, and the resulting type of the expression is `no`. An ellipsis is used not only because it will accept any argument, but also because its conversion rank is lowest, so a call to the first function will be preferred if it is possible; this removes ambiguity.

## C++11 simplification

In [C++11](https://en.wikipedia.org/wiki/C%2B%2B11), the above code could be simplified to:

```cpp
#include <iostream>
#include <type_traits>

template <typename... Ts>
using void_t = void;

template <typename T, typename = void>
struct has_typedef_foobar : std::false_type {};

template <typename T>
struct has_typedef_foobar<T, void_t<typename T::foobar>> : std::true_type {};

struct foo {
  using foobar = float;
};

int main() {
  std::cout << std::boolalpha;
  std::cout << has_typedef_foobar<int>::value << std::endl;
  std::cout << has_typedef_foobar<foo>::value << std::endl;
}
```

With the standardisation of the detection idiom in the [Library fundamental v2 (n4562)](http://en.cppreference.com/w/cpp/experimental/lib_extensions_2) proposal, the above code could be re-written as follows:

```cpp
#include <iostream>
#include <type_traits>

template <typename T>
using has_typedef_foobar_t = typename T::foobar;

struct foo {
  using foobar = float;
};

int main() {
  std::cout << std::boolalpha;
  std::cout << std::is_detected<has_typedef_foobar_t, int>::value << std::endl;
  std::cout << std::is_detected<has_typedef_foobar_t, foo>::value << std::endl;
}
```

The developers of [Boost](https://en.wikipedia.org/wiki/Boost_C%2B%2B_Libraries) used SFINAE in boost::enable_if[[3\]](https://en.wikipedia.org/wiki/Substitution_failure_is_not_an_error#cite_note-enable_if-3) and in other ways.