---
title: Deeper Dive into Anim Pose Space in UE4
tags: 
 - UE4
 - Animation
category: UE4
date: 2021-08-11 15:38:03
updated: 2021-08-12 21:49:28
description: This article tells you underlying logic about animation pose space. 
---

# Introduction 

Anim Pose Space, which is widely used across the Anim Graph. 

Generally speaking, most nodes, except `Aim Offset` and `Layered Bone Blend` node, work in `local space` if its `pose pins` are shaded white: 

![node_Rotate_Root_Bone-Base_Pose_callout.webp](node_Rotate_Root_Bone-Base_Pose_callout.webp)

 and in `component space` if its `pose pins` are shaded blue: 

![Pose Input Pins](node_Spring_Controller-Component_Pose_callout.jpg)

But what's the key difference between a `local space pose` and a `component space pose`? What is happening deep inside when a space conversion is performed? 

# Local&Component Space Pose

## Local Pose

You should override `Evaluate_AnyThread(FPoseContext& Output)` function if the node is calculated in local space. A `FPoseContext`, a struct containing a pose and a curve, is passed across animation nodes in Anim Graph Tree: 

```cpp
/** Evaluation context passed around during animation tree evaluation */
struct FPoseContext : public FAnimationBaseContext
{
public:
	/* These Pose/Curve is stack allocator. You should not use it outside of stack. */
	FCompactPose	Pose;
	FBlendedCurve	Curve;
```

This pose is in local space, which means that each bone's transformation matrix is in relation to its parent bone. 

## Component Pose

You should override `EvaluateComponentSpace_AnyThread(FComponentSpacePoseContext& Output)` function if the node is calculated in component space. 

A `FComponentSpacePoseContext` contains a `FCSPose<FCompactPose>` instead of a `FCompactPose`: 

```cpp
/** Evaluation context passed around during animation tree evaluation */
struct FComponentSpacePoseContext : public FAnimationBaseContext
{
public:
	FCSPose<FCompactPose>	Pose;
	FBlendedCurve			Curve;
```

The key difference between a `FCSPose<FCompactPose>` and a `FCompactPose`, in my opinion, is the `ComponentSpaceFlags`: 

```cpp
	// Flags to track each bones current state (0 means local pose, 1 means component space pose)
	TCustomBoneIndexArray<uint8, BoneIndexType> ComponentSpaceFlags;
```

This is an array to track each bone's current state (0 means local pose, 1 means component space pose). 

# Space Conversion

## Local To Component

`FAnimNode_ConvertLocalToComponentSpace` is used to convert pose from local space to component space: 

![image-20210812203116357](image-20210812203116357.png)

This is a cheap node because no matrix calculation is actually performed. UE4 use lazy calculation for `FCSPose<FCompactPose>`. In other words, a bone's component space transform is calculated only when needed. 

## Lazy Calculation

Think about a pose like this: 

![image-20210812205515711](image-20210812205515711.png)

It can be easily see that all bone's transform are in local space. Now, after a `FAnimNode_ConvertLocalToComponentSpace` node is evaluated, this transformation matrix is not changed at all. 

When we need to use a `Skeletal Control Node`, let's say, a `Transform (Modify) Bone` to process bone `D` in component space: 

![image-20210812205629263](image-20210812205629263.png)

During evaluation of this node, `FA2CSPose::CalculateComponentSpaceTransform` function would be called to recursively calculate component space transform of bone `D`, `C`, `B` and`A`: 

``` cpp
/** Calculate all transform till parent **/
void FA2CSPose::CalculateComponentSpaceTransform(int32 BoneIndex)
{
	check( ComponentSpaceFlags[BoneIndex] == 0 );
	checkSlow( IsValid() );

	// root is already verified, so root should not come here
	// check AllocateLocalPoses
	const int32 ParentIndex = BoneContainer->GetParentBoneIndex(BoneIndex);

	// if Parent already has been calculated, use it
	if( ComponentSpaceFlags[ParentIndex] == 0 )
	{
		// if Parent hasn't been calculated, also calculate parents
		CalculateComponentSpaceTransform(ParentIndex);
	}

	// current Bones(Index) should contain LocalPoses.
	Bones[BoneIndex] = Bones[BoneIndex] * Bones[ParentIndex];
	Bones[BoneIndex].NormalizeRotation();
	ComponentSpaceFlags[BoneIndex] = 1;
}
```



And the bone transformation matrix and `ComponentSpaceFlags` is like:  

![image-20210812210115997](image-20210812210115997.png)

As a result, space transform calculation would be performed only when needed. 

## Component To Local

As for `Component To Local` space conversion node. Bones would be traversed from leaf to root to finally get the relative transform matrix. 

# Pose Blending In Different Space

Think about two poses: 

![image-20210812211328059](image-20210812211328059.png)

![image-20210812211347973](image-20210812211347973.png)

Pose blending looks different in different space. 

## Local Space Pose Blending

In local space, pose blending is like: 

![pose_blending_local](pose_blending_local.gif)

## Component Space Pose Blending

In component space, pose blending is like: 

![pose_blending_component](pose_blending_component.gif)

# Conclusion

In order to avoid those unnecessary space-conversion calculations, please put those `Skeletal Control Node` together. 

Moreover, you can modify or add your own anim node to get rid of those space-conversion node. You can refer to [this blog](/2021/06/25/component-space-save-cached-pose-node-in-ue4)  for more info. 

And, we have our own blend list node to support blending in different spaces: 

![image-20210812213459097](image-20210812213459097.png)

You can try to implement this node because it's not difficult at all. 
