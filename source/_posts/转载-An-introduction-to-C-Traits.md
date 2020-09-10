---
title: '[转载]An introduction to C++ Traits'
date: 2020-09-10 15:14:07
tags: [Cpp,STL]
categories:
cover: https://www.modernescpp.com/images/blog/ModernCpp/CppCoreGuidelinesTypeTraitsII/TypeTraits.PNG
---
<meta name="referrer" content="no-referrer" />

Overload Journal #43 - Jun 2001 + Programming Topics  Author: [Thaddaeus Frogley](mailto:thaddaeus.frogley@creaturelabs.com)

It is not uncommon to see different pieces of code that have basically the same structure, but contain variation in the details. Ideally we would be able to reuse the structure, and factor out the variations. In 'C' this might be done by using function pointers, as in the C Standard Library `qsort` function or in C++ by using virtual functions. Unfortunately this differed to runtime what is known at compile time, and became with a runtime overhead.

C++ introduces generic programming, with `template`s, eliminating the need for runtime binding, but at first glance this still looks like a compromise, after all, the same algorithm will not work optimally with every data structure. Sorting a linked list is different to sorting an array. Sorted data can be searched much faster than unsorted data.

The C++ traits technique provides an answer.

> Think of a trait as a small object whose main purpose is to carry information used by another object or algorithm to determine "policy" or "implementation details". - Bjarne Stroustrup

Both C and C++ programmers should be familiar with `limits.h`, and `float.h`, which are used to determine the various properties of the integer and floating point types.

Most C++ programmers are familiar with `std::numeric_limits`, which at first glance simply provides the same service, implemented differently. By taking a closer look at `numeric_limits` we uncover the first advantage of traits, a consistent interface.

Using `float.h`, and `limits.h`, you have to remember the type prefix and the trait, for example, `DBL_MAX` contains the "maximum value" trait for the double data type. By using a traits class such as `numeric_limits` the type becomes part of the name, so that the maximum value for a double becomes `numeric_limits< double >::max()`, more to the point, you don't need to know which type you need to use. For example, take this simple template function (adapted from [[Veldhuizen](https://accu.org/index.php/journals/442#Veldhuizen)]), which returns the largest value in an array:

```
template< class T > 
T findMax(const T const * data, 
         const size_t const numItems) { 
  // Obtain the minimum value for type T 
  T largest = 
      std::numeric_limits< T >::min(); 
  for(unsigned int i=0; i<numItems; ++i) 
  if (data[i] > largest) 
  largest = data[i]; 
  return largest; 
} 
```

Note the use of `numeric_limits`. As you can see, where as with the C style limits.h idiom, where you must know the type, with the C++ traits idiom, only the compiler needs to know the type. Not only that, but `numeric_limits`, as with most traits, can be extended to include your own custom types (such as a fixed point, or arbitrary precision arithmetic classes) simply by creating a specialisation of the template.

But I'd like to move away from `numeric_limits`, its just an example of traits in action, and I'd like to take you through creating traits classes of your own.

First lets look at one of the simplest traits classes you can get (from [boost.org](http://www.boost.org/)) and that's the `is_void` [[boost](https://accu.org/index.php/journals/442#boost)] trait.

First, a generic template is defined that implements the default behaviour. In this case, all but one type is void, so `is_void::value` should be `false`, so we start with:

```
template< typename T > 
struct is_void{ 
  static const bool value = false;
};
```

Add to that a specialisation for void:

```
template<> 
struct is_void< void >{ 
  static const bool value = true; 
};
```

And we have a complete traits type that can be used to detect if any given type, passed in as a template parameter, is void. Not the most useful piece of code on its own, but definitely a useful demonstration of the technique.

Now, while fully specialised templates are useful and in my experiance, the most common sort of trait class specialisation, I think that it is worth quickly looking at partial specialisation, in this case, `boost::is_pointer` [[boost](https://accu.org/index.php/journals/442#boost)]. Again, a default template class is defined:

```
template< typename T > 
struct is_pointer{ 
  static const bool value = false; 
};
```

And a partial specialisation for all pointer types is added:

```
template< typename T > 
struct is_pointer< T* >{ 
  static const bool value = true; 
};
```

So, having got this far, how can this technique be used to solve the lowest common denominator problem? How can it be used to select an appropriate algorithm at compile time? This is best demonstrated with an example.

First a default traits class is created, for this example we will call it `supports_optimised_implementation`, and, other than the name, it will be the same is the `is_void` example. Next the default algorithm is implemented, inside a templated `algorithm_selector`, in this example the `algorithm_selector` template is parameterised with a bool, but in situations where a number of algorithms could be appropriate it could just as easily be parameterised with an int, or an enum. In this case `true` will mean "use optimised algorithm in object".

```
template< bool b > 
struct algorithm_selector { 
  template< typename T > 
  static void implementation( T& object ) 
  { 
//implement the alorithm operating on "object" here 
  } 
};
```

Next a specialisation of `algorithm_selector` is added which, in this case, passes the responsibility for implementing the algorithm back to the author of the object being operated on, but could well implement a second version of the operation itself.

```
template<> 
struct algorithm_selector< true > { 
  template< typename T > 
  static void implementation( T& object )   { 
    object.optimised_implementation(); 
  } 
};
```

Then we write the generic function that the end user of your algorithm will call, note that it in turn calls `algorithm_selector`, parameterised using our `supports_optimised_implementation` traits class:

```
template< typename T > 
void algorithm( T& object ) { 
  algorithm_selector< supports_optimised_implementation< T >::value >::implementation(object); 
}
```

Now all that's left to do is test it against a class that doesn't support the feature ( `class ObjectA{};` ), and a class that does:

```
class ObjectB { 
public: 
  void optimised_implementation() { 
//... 
  } 
};
```

specialisation of `supports_optimised_implementation` trait for `ObjectB`

```
template<> 
struct 
supports_optimised_implementation< ObjectB > { 
  static const bool value = true; 
};
```

Finally, instantiate the templates:

```
int main(int argc, char* argv[]) { 
  ObjectA a; 
  algorithm( a ); 
// calls default implementation 
  ObjectB b; 
  algorithm( b ); 
// calls 
// ObjectB::optimised_implementation(); 
  return 0; 
}
```

And that's it. Hopefully you can now "wow" your friends and colleague with your in-depth understanding of the c++ traits concept. :)

## Notes

It should be noted that the examples in this article require a cutting edge, standard compliant, compiler. For example, MSVC++ 6.0 does not support static constants, and will balk on:

```
static const bool value = false;
```

This particular problem can be worked around by using an enum in its place, i.e:

```
enum { value = false }; 
```

Please contact your compiler vendor for appropriate work arounds and bug fixes for your platform.

With thanks to Mark Radford, Gavin Buttimore, Martin Lester, Russ Williams & Francis Irving.

## References and Further Reading



[Myers] *Traits: a new and useful template technique* by Nathan C. Myers.



[Veldhuizen] *Using C++ Trait Classes for Scientific Computing* by Todd Veldhuizen.



[boost] [boost.org](http://www.boost.org/)'s `type_traits`.



[Maddock] *C++ Type Traits* by John Maddock & Steve Cleary, *Dr Dobb's Journal* #317



[Alexandrescu1] *Traits: The else-if-then of Types* - Andrei Alexandrescu

*Traits on Steroids* - Andrei Alexandrescu



[Sutter] Guru of the Week #71 *Inheritance Traits?* - Herb Sutter