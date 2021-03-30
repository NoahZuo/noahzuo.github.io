---
title: How To Get A Spline's Length Using Gauss–Legendre Quadrature? 
mathjax: true
tags: 
 - Math
category: Math
date: 2016-11-11 14:07:00
updated: 2021-03-30 20:29:28
description: This article tells you how to get length of a spline using Gauss–Legendre Quadrature. 
---



# Introduction

Sometimes, we need to get the length of a spline. For example, hair length is required when hair-style is modeled using Hermit splines. 

Actually this post is simply translated from the one I wrote on CSDN: https://blog.csdn.net/noahzuo/article/details/53127319



# Gauss–Legendre Quadrature

When we try to find the integration value of a function $\int_{-1}^{1}f(x) dx$, it is a good choice to use Gaussian quadrature: 

$$\sum_{i=1}^n w_i f(x_i)$$

, which  $w_i, i = 1 ... n$ is the weight to get the approximate result. 

But it is known to all that `Guass Quadrature` can only be used for functions that can be well-approximated by a polynomial on $[−1, 1]$. 

Thus, we can make $f(x) = W(x)g(x)$, which $g(x)$ is a approximated polynomial function, and $W(x)$ is a known weight function. As a result, we have: 

$$\int_{-1}^1 f(x) dx = \int_{-1}^1 W(x)g(x)dx \approx \sum_{i=1}^n w_i'g(x_i)$$

For `Gauss–Legendre Quadrature`, which needs to be sampled only 5 times to get a reasonable result,  let $P_n(x)$ be the polynomial function when $W(x)=1$. We have: 

$$ w_i = \frac{2}{(1-x_i^2)[P_n'(x_i)^2]} $$

, which $x_i$ is the $i_{th}$ root of $P_n(x)$, and $P_n(x)$ is: 

$$ P_n(x) = \prod_{1\le i\le n}(x-x_i) $$

For those 5 sample points, we have the value table: 

|$P_i$|$w_i$|
|:-----|:-------|
|$0$|$\tfrac{128}{255}$|
|$\pm\tfrac{\sqrt{245-14\sqrt{70}}}{21}$|$\tfrac{322+13\sqrt{70}}{900}$|
|$\pm\tfrac{\sqrt{245+14\sqrt{70}}}{21}$ | $\tfrac{322-13\sqrt{70}}{900}$|

# Implementation Code

As a result, all those values can be calculated in advance, so we can use it directly: 

```cpp
	struct FLegendreGaussCoefficient
	{
		float Abscissa;
		float Weight;
	};

	static const FLegendreGaussCoefficient LegendreGaussCoefficients[] =
	{
		{ 0.0f, 0.5688889f },
		{ -0.5384693f, 0.47862867f },
		{ 0.5384693f, 0.47862867f },
		{ -0.90617985f, 0.23692688f },
		{ 0.90617985f, 0.23692688f }
	};
```

So, the algorithm is truly easy. We take 4 vectors $P_0, P_1, T_0, T_1$ representing the position and tangent of starting point and ending point: 


```cpp
float LengthOfSegment(FVector P0, FVector P1, FVector T0, FVector T1)
{

	FVector Coeff1 = ((P0 - P1) * 2.0f + T0 + T1) * 3.0f;
	FVector Coeff2 = (P1 - P0) * 6.0f - T0 * 4.0f - T1 * 2.0f;
	FVector Coeff3 = T0;
	float HalfParam = 0.5f;
	float Length = 0.0f;

	for (const auto& LegendreGaussCoefficient : LegendreGaussCoefficients)
	{
		// Calculate derivative at each Legendre-Gauss sample, and perform a weighted sum
		const float Alpha = HalfParam * (1.0f + LegendreGaussCoefficient.Abscissa);
		const FVector Derivative = ((Coeff1 * Alpha + Coeff2) * Alpha + Coeff3);
		Length += Derivative.Size() * LegendreGaussCoefficient.Weight;
	}
	Length *= HalfParam;

	return Length;
}
```

Finally we have our result. 