---
title: How To Detect Intersection Between A Circle And An AABB
mathjax: true
tags: 
 - Math
category: Math
date: 2021-04-02 15:23:12
description: This article tells you how to detect intersection between a circle and an AABB. 
---

This article is simply a summary from two posts I once wrote on CSDN: https://blog.csdn.net/noahzuo/article/details/52472507, https://blog.csdn.net/noahzuo/article/details/52472507. 

# Introduction

Collision detection between an AABB and a Sphere is a common case in game physics. Thus it is important to master this algorithm in terms of performance. In this article, three methods are introduced to handle this problem. 



# Arvo’s Algorithm

## Original Algorithm

Jim Arvo has put up with a simple method to solve this problem in *Graphic Gems*, you can find the code [here](https://github.com/erich666/GraphicsGems/blob/master/gems/BoxSphere.c). 

The pseudocode is like: 

> *OVERLAPSPHEREAABB_ARVO(c, r, min, max)* 
  1. $d \leftarrow 0$
  2. for each $i \in \{ x, y, z  \}$
  3. &nbsp;&nbsp;&nbsp;&nbsp;if ($c_i < min_i$)
  4. &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;$e \leftarrow c_i - min_i$
  5. &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;$d \leftarrow d + e^2$
  6. &nbsp;&nbsp;&nbsp;&nbsp;else if ($c_i > max_i$)
  7. &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;$e \leftarrow c_i - max_i$
  8. &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;$d \leftarrow d + e^2$
  9. if ($d \leq r_2$) return true
  10. return false



This method is simple but fast as well theoretically.  But we can make it faster in practice. 

## Make It Faster

Thomas Larsson presented a faster method in practice: https://doi.org/10.1080/2151237X.2007.10129232

The key is to expand the AABB by `r` first. If `c` still locates outside the box, the test failed and returns false directly. 

As a result, we save some performance by avoiding calculations ensued. 

The pseudocode is like: 

> OVERLAPSPHEREAABB_QRI(c, r, min, max)
  1. $d\leftarrow 0$
  2. for each $i\in \{x, y, z \}$
  3. &nbsp;&nbsp;&nbsp;&nbsp;if $((e \leftarrow c_i - min_i) < 0)$
  4. &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;if $(e < -r)$ return false
  5. &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;$d \leftarrow d + e^2$
  6. &nbsp;&nbsp;&nbsp;&nbsp;else if $((e\leftarrow c_i - max_i) > 0)$
  7. &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;if $(e > r)$ return false
  8. &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;$d \leftarrow d + e^2$
  9. if ($d \leq r_2$) return true
  10. return false



This is called QRI(quick rejections interwined). Of course we can do it at the very beginning, and this variant is called QRF(quick rejections first). 



## Eliminate Conditional Branch

Even most modern computer has implemented branch prediction mechanism, we still consider a condition branch is expensive and should be avoid as much as possible. 

Thus `QRI` can be refactored as: 

> 
  1. $d\leftarrow 0$
  2. for each $i \in \{ x, y, z \}$
  3. &nbsp;&nbsp;&nbsp;&nbsp;$e \leftarrow max(min_i - c_i, 0) + max(c_i - max_i, 0)$
  4. &nbsp;&nbsp;&nbsp;&nbsp;if $(e \leq r)$ return false
  5. &nbsp;&nbsp;&nbsp;&nbsp;$d \leftarrow d + e^2$
  6. if $(d \leq r^2)$ return true
  7. return false 

As we can see, the algorithm can take good advantage of `max` instruction, which is super fast on most `SIMD` instruction set. 

## Even Faster 
So there is an even faster method: 

First, we shift the origin to the center of the rectangle. And let `a` be the vector pointing from the origin to the rectangle-corner in the first quadrant: 

![img](20160726164008951)

We flip the circle center `E` to the first quadrant `E'`by axis flipping, no matter which quadrant `E` locates. And let `b` be the vector from the origin to `E'`: 

![img](20160726164214172)

And here is the magic: Let `c` be the vector pointing from `A` to `E'`, and set `c`'s all negative element to `0`: 

- $x \geq 0, y \geq 0$: 

  ![img](20160726164539371)

- $x < 0, y > 0$: 

  ![img](20160726165238639)

- $x < 0, y \geq 0$: 

  ![img](20160726165356440)

- $x < 0, y < 0$: 

  $$c = (0, 0)$$

Then we simply compare the length of `r` and `c`: 

 - Intersection if $c < r$ 

 - Tangency if $c = r$ 


 - Separation otherwise



```cpp
bool Intersection(float2 c, float2 h, float2 p, float r) 
{
    float2 v = abs(p - c); 
    float2 u = max(v - h, 0); 
    return dot(u, u) < r * r; 
} 
```



This method can be easily expanded to higher dimensions, and is really fast. 