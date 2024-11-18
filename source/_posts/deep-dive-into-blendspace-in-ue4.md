---
title: Deep Dive into Blend Space in UE4
tags: 
 - UE4
 - Animation
category: UE4
date: 2020-10-13 10:02:51
updated: 2020-10-16 15:27:03
description: Let's take a deep look into Blend Space in UE4. Mainly focus on something that most developers does not know or notice. 
---

# Blend Space Sample Weight

Consider a blend space like this: 

![Blend Space](image-20201026212303435.png)

Each grid point contains a sample weight calculated using `[Delaunay Triangulation](https://en.wikipedia.org/wiki/Delaunay_triangulation)`, and this is calculated offline. 

![Grid Point Weight](image-20201026213607777.png)

For a given (x, y) coordinate, its final weight is calculated using bi-linear interpolation in the rectangle this coordinate locates in: 

![Bi-Linear Interpolation](image-20201026214207321.png)



# Play Length&Play Rate

You may see varying animatino length if you call `UBlendSpaceBase::GetAnimationLengthFromSampleData` because the `BlendSpace` node is synchronizing playback of the source animation clips within it. 

For example, if you have one animation clip that is 1.0 seconds in length being blended in a `BlendSpace` with another clip that is 0.5 in length, when the weight of the `BlendSpace` is 50% between the two source clips, the value returned from `GetAnimationLengthFromSampleData()` will be 0.75.  

![image-20201027212612568](image-20201027212612568.png)

The playback rate of each animation clip is then scaled to match this weighting. Consider a `BlendSpace` like this: 

![image-20201027213845268](image-20201027213845268.png)

If the delta time of current frame is 0.25s. In order to synchronize all animations, the playback rate of each animation clip is scaled in accordance with the normalized delta time. 



Oh and by the way, each blend space holds the normalized time as when dealing with changing anim length, it would be possible to go backwards. As a result, the playback of each animation clip is set by: 

```cpp
CurrentSampleDataTime = SampleNormalizedCurrentTime * Sample.Animation->SequenceLength;
```



# Sync Marker in a `BlendSpace`

Sync marker in a blendspace is more complicated, compared to sync marker in an animation sequence. 

In a blendspace, it will still sync even if only one node in sync group. So you're never non-sync group unless you have situation where some markers are relevant to one sync group but not all the time. 

Here we save NormalizedCurrentTime as Highest weighted samples' position in sync group. If you're not in sync group, NormalizedCurrentTime is based on normalized length by sample weights. If you move between sync to non sync within blendspace, you're going to see pop because we'll have to jump. 

For now, our rule is to keep normalized time as highest weighted sample position within its own length. Also MoveDelta doesn't work if you're in sync group. It will move according to sync group position. 

