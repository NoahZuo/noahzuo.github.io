---
title: A Simple Trick That Save Some Anim Performance In UE4
tags: 
 - UE4
 - Animation
category: Animation
date: 2020-02-17 15:47:12
description: This article introduces a trick that can save some performance for anim-evaluation in UE4. 
---

## Introduction

The optimization is about the animation node `Blendlist_Base`, which means all blend list nodes(eg. `Blend Pose By Bool`, `Blend Pose By Enum` and `Blend Pose By Integer`) can benefit from this optimization. 

![Blend Poses](blend-poses.png)

## Performance Waste
The key point of this optimization is, in most of the times, only one branch would be evaluated for a blendlist. 

![Evaluated Branch](Evaluated-Branch.png)

But in `FAnimNode_BlendListBase::Evaluate_AnyThread`, a `FAnimationRuntime::BlendPosesTogether` function would be called even only one branch is evaluated. 

```cpp
		for (int32 i = 0; i < PosesToEvaluate.Num(); ++i)
		{
			int32 PoseIndex = PosesToEvaluate[i];

			FPoseContext EvaluateContext(Output);

			FPoseLink& CurrentPose = BlendPose[PoseIndex];
			CurrentPose.Evaluate(EvaluateContext);

			FilteredPoses[i].MoveBonesFrom(EvaluateContext.Pose);
			FilteredCurve[i].MoveFrom(EvaluateContext.Curve);
		}

		// Use the calculated blend sample data if we're blending per-bone
		if (BlendProfile)
		{
			FAnimationRuntime::BlendPosesTogetherPerBone(FilteredPoses, FilteredCurve, BlendProfile, PerBoneSampleData, PosesToEvaluate, Output.Pose, Output.Curve);
		}
		else
		{
			FAnimationRuntime::BlendPosesTogether(FilteredPoses, FilteredCurve, BlendWeights, PosesToEvaluate, Output.Pose, Output.Curve);
		}		
```

As a result, `BlendPose<ETransformBlendMode::Overwrite>` would be called, and this leads to a per bone blend: 

![Per Bone Blend](per-bone-blend.png)

## How To Fix
It is fairly easy to fix this issue: We simply Evaluate the branch to the `Output` context when only one branch is valid. 

![How to fix](how-to-fix.png)
