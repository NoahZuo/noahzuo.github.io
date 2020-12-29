---
title: Tool tips for ue4 animation programming
tags: 
 - UE4
 - Animation
 - Programming
category: UE4
date: 2019-05-11 13:40:12
description: This article tell you some issue you might encounter while doing animation development in UE4
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



### Don't use `TMap` for your own `AnimNode`

Well... at lease not in 4.20
Using `TMap` causing crash when hitting compile button or save. 

```cpp
// FAnimBlueprintCompilerContext::CreateEvaluationHandlerStruct
// Does it get serviced by this handler?
		if (FAnimNodeSinglePropertyHandler* SourceInfo = Record.ServicedProperties.Find(PropertyName))
		{
			if (TargetPin->PinType.IsArray())
			{
				// ... 
			}
			else
			{
				// Causing crash here. 
				check(!TargetPin->PinType.IsContainer())
				// ...
			}
		}
```



### Watch out out bone order in your `EvaluateSkeletalControl_AnyThread` function. 

See `FCSPose<PoseType>::LocalBlendCSBoneTransforms` function. There is a *Parents before Children* order check in this function. 

```cpp
template<class PoseType>
void FCSPose<PoseType>::LocalBlendCSBoneTransforms(const TArray<struct FBoneTransform>& BoneTransforms, float Alpha)
{
	SCOPE_CYCLE_COUNTER(STAT_LocalBlendCSBoneTransforms);

	// if Alpha is small enough, skip
	if (Alpha < ZERO_ANIMWEIGHT_THRESH)
	{
		return;
	}

#if DO_CHECK
	if (BoneTransforms.Num() > 0)
	{
		FCompactPoseBoneIndex LastIndex(BoneTransforms[0].BoneIndex);
		// Make sure bones are sorted in "Parents before Children" order.
		for (int32 I = 1; I < BoneTransforms.Num(); ++I)
		{
			check(BoneTransforms[I].BoneIndex >= LastIndex);
			LastIndex = BoneTransforms[I].BoneIndex;
		}
	}
#endif
    ...
```

As a result, do add following code to the end of your function: 

```cpp
OutBoneTransforms.Sort(FCompareBoneTransformIndex());
```

### How to get curve data in an `Anim Sequence`? 
Before Version 4.22: 
```c++
	const auto& curveData = inAnimSeq->GetCurveData();

	auto curveUID = inAnimSeq->GetSkeleton()->GetUIDByName(USkeleton::AnimCurveMappingName, inCurveName);

	if (curveUID == SmartName::MaxUID)
	{
		success = false;
		return errorResult;
	}
	const FFloatCurve* curve = (const FFloatCurve*)curveData.GetCurveData(curveUID, ERawCurveTrackTypes::RCT_Float);

	success = true; 
	float result = curve->Evaluate(inTime);
	return result; 
```



After Version 4.22: 

```c++
	auto curveUID = inAnimSeq->GetSkeleton()->GetUIDByName(USkeleton::AnimCurveMappingName, inCurveName);

	if (curveUID == SmartName::MaxUID)
	{
		success = false;
		return errorResult;
	}

	return inAnimSeq->EvaluateCurveData(curveUID, inTime);

```



### AnimGraphNode Module
Your `AnimGraphNode` should be placed in a `Developer` or `Uncooked` module. Otherwise it cause error in your dedicated server. 