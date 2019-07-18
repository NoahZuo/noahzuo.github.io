---
title: GJK(Gilbert-Johnson-Keerthi) algorithm for collision detection 
mathjax: true
tags: 
 - Game Physics
 - Math
category: Physics
description: This article tells you how GJK algorithm is applied for collision detection. 


---



## What is GJK Algorithm? 

GJK algorithm is proposed by Gilbert, Johnson and Keerthi for determining intersection between two polyhedra. It is one of the most effective methods. 



As originally described, GJK is a simplex-based descent algorithm that, given two sets of vertices as iputs, finds the Euclidean distance(and closest points) between the convex hulls of these sets. 



But actually the GJK algorithm can also be applied to arbitrary convex point sets, not just polyhedra. 



An important point is that the GJK algorithm does not actually operate on the two input objects per se, but on the Minkowski difference between the objects. In short, GJK algorithm searches the Minkowski difference object iteratively, a subvolume at a time, each such volume being a simplex. 

## Minkowski Difference and Sum

$$\begin{equation}
\begin{aligned}
a &= b + c \\
  &= d + e + f + g \\
  &= h + i
\end{aligned}
\end{equation}\label{eq2}$$