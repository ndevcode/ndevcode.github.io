---
layout: post
title: "Understanding the Direction of std::is_convertible in C++"
author: N Dev
---

# Understanding the Direction of std::is_convertible in C++

## Introduction

While practising templates I implemented a small array class. My objective was
to allow assigning an array containing one type to an array containing another
type, provided that the two element types are convertible.

So for example, assigning an array of integers to an array of doubles should be
allowed. Assigning an array of strings to an array of doubles should result in
a readable error at compile time.

I used `std::is_convertible` inside a `static_assert` for this check. The code
compiled and all my tests passed. However the arguments to `std::is_convertible`
were in the wrong order.

## The Array Implementation

The initial implementation was as below.

```
template <typename T, int N>
class MyArray {
  public :
     MyArray() = default;

     template <typename T2>
     MyArray<T,N>& operator=(const MyArray<T2,N>& arr) {
       static_assert(std::is_convertible<T,T2>::value,"Can't convert src to dest");
       elem_.clear();
       elem_.insert(elem_.begin(),arr.elem_.begin(),arr.elem_.end());
       return *this;
     }

     std::vector<T> elem_;
};
```

Here `T` is the element type of the destination array and `T2` is the element
type of the source array.

I tested the assignment operator using the following cases.

```
MyArray<double, 10> doubleMyArray{};
MyArray<int, 10> intMyArray{};
MyArray<std::string, 10> strMyArray{};

doubleMyArray = intMyArray;   //compiles
doubleMyArray = strMyArray;   //static_assert fails
```

Both results were as expected. So I assumed that the `static_assert` was
correct.

## The Direction of std::is_convertible

The type trait is defined as

```
std::is_convertible<From,To>
```

Note that the source type is the first argument and the destination type is the
second argument.

In the assignment operator the source is `T2` and the destination is `T`.
Therefore the check should be

```
static_assert(std::is_convertible<T2,T>::value,"Can't convert src to dest");
```

The original code used

```
std::is_convertible<T,T2>
```

which checks whether the destination can be converted to the source. This is
the opposite of the operation performed by the assignment operator.

## Why the Initial Tests Pass

The first test assigns an `int` to a `double`. C++ allows conversion in both
directions between these two types.

```
std::is_convertible<int,double>::value     //1
std::is_convertible<double,int>::value     //1
```

The second test attempts to assign a `std::string` to a `double`. Neither type
is convertible to the other.

```
std::is_convertible<std::string,double>::value     //0
std::is_convertible<double,std::string>::value     //0
```

So reversing the arguments does not change the result for either test. Adding
more tests of the same kind would not help in finding the problem.

## Using a Conversion That Works in Only One Direction

To test the direction we need two types for which the conversion is
asymmetric. `const char*` and `std::string` provide such an example.

```
std::cout << std::is_convertible<const char*,std::string>::value; //1
std::cout << std::is_convertible<std::string,const char*>::value; //0
```

A `const char*` can be used to construct a `std::string`. A `std::string`,
however, is not implicitly convertible to a `const char*`.

The results for the two versions of the check are as below.

| Assignment | Expected | `is_convertible<T,T2>` | `is_convertible<T2,T>` |
|---|---|---|---|
| `int` to `double` | Compiles | Compiles | Compiles |
| `std::string` to `double` | Fails | Fails | Fails |
| `const char*` to `std::string` | Compiles | Fails | Compiles |
| `std::string` to `const char*` | Fails | Passes the check | Fails |

The backwards check rejects the valid conversion from `const char*` to
`std::string`. It also allows the invalid conversion from `std::string` to
`const char*` to continue past the `static_assert`.

In the second case compilation still fails later when `std::vector` attempts
the assignment. With g++ 13 this produced 27 lines of errors from inside the
standard library. The purpose of adding the `static_assert` was to replace such
an error with a short readable message.

## Conclusion

`std::is_convertible` is directional. The first argument is the source type
and the second argument is the destination type. Since both arguments are
types, swapping them still produces valid C++ and the compiler cannot indicate
that the order is wrong.

A test for a directional operation should use at least one case where reversing
the direction changes the result. In this case `const char*` and `std::string`
make the incorrect ordering visible, while `int` and `double` do not.
