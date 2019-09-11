---
title: Sweep and Prune for Scene Collision Detection
mathjax: true
tags: 
 - Physics
 - Programming
 - PhysX
category: Unity
description: This article tells you something about SAP(sweep and prune) for scene collision detection.  


---

## Introduction

When we play with physX or bullet, we always need to determine which broad phase algorithm we use. The most common algorithm is `SAP`, whose full name is `sweep and prune`. Currently Unreal Engine 4 and Unity use `SAP` by default. 

Since `SAP` is one of broad phase algorithms,  the very first thing we need to know is what `Broad Phase` algorithm is. 

## Broad Phase

### Naive collision detection algorithm

I believe every one know the most naive way to detect a two-phase collision in a scene: We detect collision between each collider using a two nested loop: 

```cpp
for collider1 in allColliders: 
	for collider2 in allcolliders: 
		if collider1 == collider2: continue; 
		else: 
			detect collision between collider and collider2
```

There is no need to talk about how naive this method is. As the complexity is $O(n^{2})$, performance is totally not acceptable as collider number grows. 

Well... we still need to talk about it. Firstly, $n\*n$ collision detect results need to be calculated, second, we need to calculate those results by brute force even if no collider moves. Finally, physics system always calculates using a fixed timestep. This means we need to calculate those results multiple times even if collider only moves in one frame. 

![Naive Collision Detection](naive-collision-detection.png)

### Nowadays collision detection algorithm

In current physics engine, collision detection is split into multiple stages. We always split it into `broad phase` and `narrow phase` stage. 

![Stage Split](stage-split.png)

`Broad phase` is used to try to exclude those pairs of colliders that never collides with each other. For example, a guy in Beijing and a guy in New York. We never need to calculate whether they collides with each other because there exists no possibility for that. 

It needs to be concerned that **it is not our goal to exclude all non-collide pairs, we just try to exclude those pairs that are most not likely to collide with each other **. 

After `Broad phase`, lots of pairs that are **most likely** to collide are left. We do those actual collision detection calculation in `Narrow phase` stage. 

So that is how nowadays physics system works. 

## SAP Basic Idea

The basic idea behind the algorithm is: 

 >  two AABBs overlap **iff** their projections on the X, Y and Z coordinate axes overlap. 



