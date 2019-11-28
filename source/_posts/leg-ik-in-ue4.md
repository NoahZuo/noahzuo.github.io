---
title: Leg IK in UE4
tags: 
 - UE4
 - Animation
 - IK
category: UE4
description: This article briefly introduce the Leg IK in UE4, mainly focusing on the underlying logic. 
---

It seems that Leg IK has been existed in UE4 for quite a long time, and it seems necessary to dig into the code and try to understand what is going on underneath. 

## Leg bones

Since this node is designed for multi-bone legs IK calculation, but how does it handle multi-bone legs? `CCD` or `FABRIK`? Or some other black magic? 

```cpp
void FIKChain::ReachTarget(const FVector& InTargetLocation, float InReachPrecision, int32 InMaxIterations)
{
	if (!bInitialized)
	{
		return;
	}

	const FVector RootLocation = Links.Last().Location;

	// If we can't reach, we just go in a straight line towards the target,
	if ((NumLinks <= 2) || (FVector::DistSquared(RootLocation, InTargetLocation) >= FMath::Square(GetMaximumReach())))
	{
		const FVector Direction = (InTargetLocation - RootLocation).GetSafeNormal();
		OrientAllLinksToDirection(Direction);
	}
	// Two Bones, we can figure out solution instantly
	else if (NumLinks == 3 && (CVarAnimLegIKTwoBone.GetValueOnAnyThread() == 1))
	{
		SolveTwoBoneIK(InTargetLocation);
	}
	// Do iterative approach based on FABRIK
	else
	{
		SolveFABRIK(InTargetLocation, InReachPrecision, InMaxIterations);
	}
}
```

It is easy to tell that for a three-bone(thigh->leg->foot) leg, this node would drawback to a simple two-bone IK. 

But as for a leg that has more than 3 bones(spider leg or octopus leg), it would perform a [FABRIK](http://www.andreasaristidou.com/FABRIK.html). Since I've already wrote a [blog about FABRIK on csdn](https://blog.csdn.net/noahzuo/article/details/80188366) and I am not going to rewrite it again. 

## How does Leg IK work? 

You can define your legs in this node. Each leg can be defined using `FKFoot Bone` and `Num Bones in Limb`, which defines a leg from toe to hip. 

And you need an `IKFoot Bone` in your skeleton if you want to use this node. And during evaluation, this node would try to put your toe(in another word, `FKFoot Bone`) to the `IKFoot Bone`. 

let's check out the whole algorithm: 

1. Leg transforms would firstly be aligned with the ik target. Delta normal is calculated between `InitialDir(From Hip to FootFKLocation)` and `TargetDir(From Hip to FootIKLocation)`. 
2. After being aligned with `IKFoot Bone`, try to reach for ik effector. Either `Two bone IK` or `FABRIK` is used for this purpose. For a `Two bone IK`, `Hinge Rotation Axis` is used for bending direction, and for a `FABRIK`, `Min Rotation Angle` is used for rotation limit, if enabled. 
3. Knee twist would be adjusted if `Enable Knee Twist Correction` is checked. You need to define the `Foot Bone Forward Axis` for this feature. This axis is used for determining how much angle the foot has been twisted. By `Bone Forward Axis` it means **the axis vertical to the ground**. 