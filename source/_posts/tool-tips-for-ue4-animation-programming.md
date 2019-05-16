---
title: Tool tips for ue4 animation programming
tags: 
 - UE4
 - Animation
 - Programming
category: UE4
---

### AnimNode not evaluating your data? 

If you have your own Animation Node and find it not evaluate your input data? 

In your `Update_AnyThread` function, be sure to call: 

```cpp
GetEvaluateGraphExposedInputs().Execute(Context)
```

Caution: This shall cost some performance! 



### `FCSPose::ConvertBoneLocalSpace` Function

Bone hierarchy is an array-based tree. Thus if you want to convert all bone to local space, be sure to traverse from end to the beginning. 

