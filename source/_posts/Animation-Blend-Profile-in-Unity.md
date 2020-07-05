---
title: Animation Blend Profile in Unity
date: 2020-07-03 17:42:56
tags:
 - Unity
 - Animation
category: Unity
description: This article simply record how I implemented Animation Blend Profile in Unity Engine. 
---

# What is `Animation Blend Profile`? 
[`Animation Blend Profile`](https://docs.unrealengine.com/en-US/API/Runtime/Engine/Animation/UBlendProfile/index.html) is a UE4 animation system feature that enables us to set blend speed of each bone separately. 

This is majorly used with movement transitions which needs to avoid foot sliding and to have a fluent upper body transition. 

Transitions like `Idle<->Moving` or `Floating->Landing` need this feature desperately. Because players don't like foot sliding around, but the upper body would look bad if we set the transition time to a small value. 

Thus we need to make upper body to blend slower, and lower body to blend faster. 



# How to use animation blend profile in Unity? 

Firstly, there shall be a place for us to set blend speed of each bone. Thus, an `AvatarMask` is a wonderful option. I've add a `blend profile speed` for each bone in Unity's `AvatarMask`: 

![Avatar Mask](avatarmask.png)

It needs to be concerned that you are guaranteed to have `bone blend speed` larger than 1, otherwise there would be some problem with transition time. 

And then we can set this `blend profile AvatarMask` for each state transtition like this: 

![State Transition](transition.png)

So it is easy to know that `joint2` would be blended faster than `joint3`: 

![Blend Profile](blend_profile.gif)

