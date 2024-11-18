---
title: Optimize C++ Using Template
tags: 
 - C++
category: C++
date: 2016-04-12 15:10:03
updated: 2021-03-31 15:12:28
description: This article tells you how to optimize c++ code using template. 
---

This post is simply translated from the one I wrote on CSDN: https://blog.csdn.net/noahzuo/article/details/51133525

# Introduction

Algorithms can be templated if input parameters can be determined during compilation. As a result, compiler can do the optimization work for us during handling templates. 



# Fibonacci Sequence

We all know how to write fibonacci in a traditional but inefficient way: 

```cpp
unsigned int fib(unsigned int n) {
    if (n == 0 || n == 1)
    {
        return 1 ;
    }
    else
    {
        return fib( n - 1) + fib (n - 2); 
    }
}
```

We can use template programming to optimize it: 

```cpp
template <unsigned int N>
struct FibR 
{ 
    enum 
    { 
        Val = FibR< N-1 >:: Val + FibR::Val 
    }; 
};

template <>
struct FibR <0> 
{ 
    enum 
    { 
        Val = 1 
    };
}; 

template <>
struct FibR <1> 
{ 
    enum 
    { 
        Val = 1 
    };
};
#define fib(n) FibR::Val
```

It needs to be mentioned that a template function is a enumerated value that is calculated during compilation instead of a real function. 

For the compiler, it runs like: 

```cpp
fib (4)
= FibR< 4 >::Val
= FibR< 3 >::Val + FibR< 2 >::Val
= FibR< 2 >::Val + FibR< 1 >::Val + FibR< 1 >::Val + FibR< 0 >::Val
= FibR< 0 >::Val + FibR< 1 >::Val + FibR< 1 >::Val + FibR< 1 >::Val + FibR< 0>:: Val
= 1 + 1 + 1 + 1 + 1
= 5
```

Thus this value is solved during compilation, we can save the performance for runtime. 

# Factorial

The traditional way is like: 

```cpp
unsigned int fact (n ) 
{ 
    return n <= 1 ? 1 : n * fact( n - 1); 
}
```

But we can use template programming to optimize it: 

```cpp
template < unsigned int N >
struct FactR
 { 
    enum { 
        Val = N * FactR::Val 
    }; 
};

template <> 
struct FactR < 1 >
{
    enum 
    {
        Val = 1
    };
}; 

#define fact(n) FactR::Val
```

So if `n` can be given during compilation stage, `fact(n)` can be solved right away. 





# Drawback

Of course something needs to be mentioned: 

- We might have to suffer from longer compiling time. 
- It might be harder to read or debug the relevant code. 

