---
title: Blend to Target In Unity
tags: 
 - Unity
 - Animation
category: Unity
date: 2021-03-05 11:15:12
updated: 2021-03-05 17:44:28
description: This article introduces a feature that I implemented called `blend to target` in Unity Engine, which is widely used in `Cross File Mobile` and `Call of Duty Mobile`. 

---

# Introduction

So... blend tree in Unity is somehow rudimentary compared with blend space in UE4. And one of the most powerful feature a blend tree lacks, in my opinion, is `Target Weight Interpolation`: 

![image-20210306115441037](image-20210306115441037.png)

This is a brilliant feature, especially for character locomotion. As a result, it seems necessary to implement this feature in Unity. 

# What Is `Blend To Target`? 

Think about a blend tree that takes moving angle as its parameter: 

![image-20210306142721918](image-20210306142721918.png)

If we want the character to move from left to right, we need to lerp `Angle` parameter value from `-90`  to `90`. That is to say, this value will go from `-90` to `0`, then from `0` to `90`：

As we can imagine, the weight of `Fwd` node is affected when the chracter move from left to right. And this always looks terrible: 

![disabled](disabled.gif)

This looks even worse because the character leans forward when moving forward. 



In conclusion, `Fwd` nodes' weight should not be affected during such a transition. And this is what `Blend to target` does, and it eventually looks like this: 

![enabled](enabled.gif)

# How does it work? 

Instead of blending animations by parameter value, `Blend to target` blends animations by sample weights directly.  



When the character is moving left, the sample weights of this blend tree is `(1.0, 0.0, 0.0)`, when the character is moving right, the sample weights is `(0.0, 0.0, 1.0)`. 



Lerp between this sample weights would be performed to get the final result: 


| Progress | Left Weight | Fwd Weight |  Right Weight  |
| :----:| :----: | :----: | :----: |
| 0% | 1.0 | 0.0 | 0.0 |
| 25% | 0.75 | 0.0 | 0.25 |
| 75% | 0.25 | 0.0 | 0.75 |
| 100% | 0.0 | 0.0 | 1.0 |

And we can see the `Fwd` node is not affected now. 






