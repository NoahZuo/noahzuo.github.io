---
title: 计算机与数学 —— 检测圆与矩形相交的快速判定算法
date: 2016-07-26 17:07:40
mathjax: true
tags:
 - Math
 - Performance
 - Physics
category: Math
description: 这篇博客介绍了如何快速检测圆与矩形是否相交的算法。
---

## 算法介绍

首先，对于矩形来说，将坐标系原点移到矩形的中心，并且将从原点到第一象限的顶点的向量$a$求出来。

![image 01](image01.png)

其次，无论圆心$E$在哪一个象限，都将其通过轴对称转换到第一象限，并且求出原点到转换后的圆心$E'$的向量$b$。

![image 02](image02.png)

接下来是最关键的一步：算出$A$到$E'$的向量$c$，并且把向量$c$中的负值全部置为0，得到最终的向量$c$。

 - 两个分量都大于0

 ![image 03](image03.png)

 - $x$值小于0

 ![image 04](image04.png)

 - $y$值小于0

 ![image 05](image05.png)

$x$和$y$都小于0时，最终的向量为$(0, 0)$。

因此最终只要比较半径$r$与向量$c$的模长的大小即可，如果需要在代码中实现，则无需开方。

 - 若$c<r$，则圆与矩形相交。
 - 若$c=r$，则圆与矩形相切。
 - 否则二者相离。

## 代码实现

```cpp
bool Intersection(float2 c, float2 h, float2 p, float r) 
{
	float2 v = abs(p - c); 
	float2 u = max(v - h, 0); 
	return dot(u, u) < r * r; 
} 
```

## 思考
这种算法可以扩展至高维。例如需要判定Box与球形是否相交，则可以依次算法类推至三维的判定。

这类问题通常都可以转化为闵可夫斯基和的问题，[传送门](https://en.wikipedia.org/wiki/Minkowski_addition)。