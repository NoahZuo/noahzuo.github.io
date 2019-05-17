---
title: Tool tips for ue4 animation programming
tags: 
 - UE4
 - Animation
 - Programming
category: UE4
---

This article mainly focus on some tool-tips you might encounter while doing some animation development. And I will keep updating this article from time to time. 



### AnimNode not evaluating your data? 

If you have your own Animation Node and find it not evaluate your input data? 

In your `Update_AnyThread` function, be sure to call: 

```cpp
GetEvaluateGraphExposedInputs().Execute(Context)
```

Caution: This shall cost some performance! 



### `FCSPose::ConvertBoneLocalSpace` Function

Bone hierarchy is an array-based tree. Thus if you want to convert all bone to local space, be sure to traverse from end to the beginning. 

