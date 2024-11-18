---
title: 2D Blend Types in Unity
tags: 
 - Unity
 - Animation
category: Unity
date: 2019-05-23 21:45:12
description: This article give details about 2D blend types in Unity animation blend tree node. 

---

Unity has already provided [documentations about 2D blend types of Blend Tree](https://docs.unity3d.com/Manual/BlendTree-2DBlending.html). But frankly speaking, the documentation delivers too little information. 

This article will give more details about those blend types. Our editor images will not be provided because of confidentiality. 

## What is a Blend Type? 

Blend type serves as a *weight calculator* in Unity. Different blend type results in different weight if a 2D parameter is given. The different 2D Blend Types have different uses that they are suitable 
for. They differ in how the influence of each motion is calculated. 

I am pretty sure that some developers are confused about differences between those 2D Blend Types. And I was once one of them... 

So I'll provide detailed algorithms for each blend type. Hopefully, this will be helpful to you. 

## Simple Directional

Let's take a look at how Unity tell us about Simple Directional: 

> **2D Simple Directional**: Best used when your motions represent different directions, such as 
> “walk forward”, “walk backward”, “walk left”, and “walk right”, or “aim 
> up”, “aim down”, “aim left”, and “aim right”. Optionally a single motion
> at position (0, 0) can be included, such as “idle” or “aim straight”. 
> In the Simple Directional type there should *not* be multiple motions in the same direction, such as “walk forward” and “run forward”.

### If the blend position is *(0, 0)*

**A motion position in (0, 0) makes a huge difference. **

If your blend position is *(0, 0)*, and there is a motion field at position *(0, 0)*. Then this motion field would be full-weight. 

![Simple Directional](simple directional.png)

If there is not a motion at position *(0, 0)*. Then all motion would have an averaged weight. No matter what the motion position is. 

![Simple Directional 2](simple directional2.png)

Surprised, huh? Some developers might believe that a motion closer to the origin point would have a larger weight, but actually... it's not. 

**Also... Do not place more than 1 motion in position *(0, 0)*, function would directly return and all weight would be 0.** 

### If the blend position is not *(0, 0)*

If the blend position is not *(0, 0)*, all motion field would be traversed. The clockwise-closest and anti-clockwise-closest motion field(A and B) would be found. 

![Simple directional 5](simple directional5.png)

And a triangle with points (A, B and origin point) would be formed, weight of each point would be calculated through [Barycentric Coordinate System](https://en.wikipedia.org/wiki/Barycentric_coordinate_system). 

#### If there is a motion field in position *(0, 0)*

This would be the common case. Barycentric coordinate of triangle points would be the weights of these three motion fields. 

![Simple Directional 3](simple directional3.png)

#### If there is not a motion field in position *(0, 0)*

If there is not a motion field in position *(0, 0)*. Weight of point *A* and *B* would be kept, and the weight of the point *O* would be given to every point. 

That means, a shared weight would be add to every motion field (including point A and B). 

![Simple Directional 4](simple directional4.png)

## Freeform Directional

The Freeform stuff is much more complicated. And frankly speaking, it is too complicate to describe the whole algorithm in this article. 

Well... In short, every motion field has a neighbor list which contains what is so called *neighborhoods* of each  motion field. 

Unity has its complicate algorithm about weight&neighborhood calculation. And I am not sure whether I am allowed to write it down in an article. Let's make vector from the origin to the blend position *v0*, and make vectors from the origin to the motion fields *v1*,*v2*, *v3* and *v4* like this: 

![Freeform Directional](Freeform Directional.png)

In general, the final result heavily depends on not only the angle between vectors, but also the magnitude of each vector. As a result, this blend type allows multiple existences of motion field in the same direction: 

![Freeform Directional 2](Freeform Directional2.png)

## Freeform Cartesian

Freeform Cartesian blend type also has neighbor list stuff. But Unity does not care about angle with this blend type. Instead, only positions are concerned. 

This is much the same as 2D Blend Space in Unreal Engine 4. And it is the most frequently-used blend type in our Unity project. 



## Performance 

As for performance, the `simple directional` should be the cheapest and the `freeform directional` should be the most expensive. You can choose what you need most to achieve what you want. 

