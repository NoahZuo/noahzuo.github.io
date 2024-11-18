---
title: A Tiny Performance Issue In UE4 Camera View
tags: 
 - UE4
 - Performance
category: UE4
date: 2022-01-18 15:15:27
updated: 2022-01-18 15:15:27
description: This article introduces a tiny performance issue about camera view that I've encountered in UE4. 
---

# Issue Introduction

I've been playing with `streamlines` for quite a long time. And actually it helps us a lot to find our performance issue. 

In our project, camera view point information is required in a lot of places. By default we use function like `APlayerCameraManager::GetCameraViewPoint` to get exactly what we need. 

However we've found that a strange function cost: 

![image-20220118153154176](image-20220118153154176.png)

So it seems that a copy construction of `FMinimalViewInfo` is performed here: 

![image-20220118153353029](image-20220118153353029.png)

And yes, there is a `FPostProcessSettings`, a complex struct, inside a `FMinimalViewInfo`. And this can cost a considerable CPU performance. 

# Issue Solution

Well we simply avoid copy construction of `FMinimalViewInfo` and directly return those result required. This can save some performance. 



# An Irrelevant Note

Well... I just need a place to write this down since I always forget this feature. 

If you want a detailed performance information about slate performance. You can visit `slateglobals.h` to find more information. 