---
title: Leg IK in UE4
tags: 
 - UE4
 - Animation
 - IK
category: UE4
date: 2019-11-28 21:39:33
description: This article briefly introduce the Leg IK in UE4, mainly focusing on the underlying logic. 
---

It seems that Leg IK has been existed in UE4 for quite a long time, and it seems necessary to dig into the code and try to understand what is going on underneath. 

And btw, it seems to be very convenient to use [geogebra](https://www.geogebra.org/3d?lang=en) to draw 3D diagrams. This article would use diagrams created by this website. 

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

It is easy to tell that for a three-bone(thigh->leg->foot) leg, this solver would eventually fallback to a simple two-bone IK. 

But as for a leg that has more than 3 bones(spider leg or octopus leg), it would perform a [FABRIK](http://www.andreasaristidou.com/FABRIK.html). Since I've already wrote a [blog about FABRIK on csdn](https://blog.csdn.net/noahzuo/article/details/80188366) and I am not going to rewrite it again. 

## How does Leg IK work? 

You can define your legs in this node. Each leg can be defined using `FKFoot Bone` and `Num Bones in Limb`, which defines a leg from toe to hip. 

And you need an `IKFoot Bone` in your skeleton if you want to use this node. And during evaluation, this node would try to put your toe(in another word, `FKFoot Bone`) to the `IKFoot Bone`. 

!![initial](initial.png)

let's check out the whole algorithm: 

1. Leg transforms would firstly be aligned with the ik target. Delta normal is calculated between `InitialDir(From Hip to FootFKLocation)` and `TargetDir(From Hip to FootIKLocation)`. 

![align-legs](align-legs.png)

2. After being aligned with `IKFoot Bone`, try to reach for ik effector. Either `Two bone IK` or `FABRIK` is used for this purpose. For a `Two bone IK`, `Hinge Rotation Axis` is used for bending direction, and for a `FABRIK`, `Min Rotation Angle` is used for rotation limit, if enabled. 
3. Knee twist would be adjusted if `Enable Knee Twist Correction` is checked. You need to define the `Foot Bone Forward Axis` for this feature. This axis is used for determining how much angle the foot has been twisted. By `Bone Forward Axis` it means **the axis pointing to the ground**. 

![footboneforwardaxis](footboneforwardaxis.png)

## Rotation limit for FABRIK

Something still needs to be done for constraints in FABRIK, when used for leg ik calculation. And hence we would go through those extra work here. 

### Plane normal&LinkAxisZ

Since we have just mentioned that legs would be aligned to the target effector, thus bone rotation limits are calculated in a 2-D space. 

![limit-in-plane](limit-in-plane.png)

First thing that needs to be done is to get a valid plane normal for the whole leg. 

By traverse vector`A->B'`, `A->C'` and `A->FKFootBone'` until we get a valid cross product result with vector `A->IKFootBone`. 

![planeNormal](planeNormal.png)

After getting a valid plane normal, we still need a valid `LinkAxisZ` vector for each leg bone. The reason is that sometimes we may have a leg inner angle larger than `pi`. 

![strange-leg](strange-leg.png)

In this case, we need to flip plane normal, and this is what `LinkAxisZ` do during limiting rotation, See `LinkAxisZVector` in the following image: 

![flipping-plane-normal](flipping-plane-normal.png)

### Apply rotation limitation

For a simple FABRIK without rotation limitation, the whole leg would loot like this after a forwardpass: 

![forwardpass](forwardpass.png)

As for a leg, it looks terrible. Thus we need to apply rotation limit to it. Rotation limit is applied after each bone's reaching stage, both forward and backward. 

Let's take a look at bone `C''` rotation limit as an example: 

1. First `childAxisYC''` is calculated using  a cross product between `LinkAxisZ` and `childAxisXC''`, which point from `C''` to its child bone. 

![childAxisYC](childAxisYC.png)

2. Then dot product between `parentAxisC''`, which points from `C''` to its parent bone, and `childAxisXC''` and `childAxisYC''` is calculated. By this we could get the `cos` and `sin` value of current angle.   We use this to determine whether current angle has already reached the limit. 

![angle-limit](angle-limit.png)

3. If the  `sin` value if less than `0`(which means the the leg inner angle is larger than `pi` and the leg needs to be flipped), or the `cos` value is larger than the `cos(MinRotationAngleRadians)`(which means that the angle has already reached the angle limit), a rotation limitation should be performed. The current angle in the image above is `56.15` degrees, and our `MinRotationAngleRadians` is 60 degrees, which is larger than current degree. 

4. As a result, we need to apply rotation limitation now. Since this is a `forward reach pass`, we need to adjust the parent bone location by code like this:

   ``` 
   ParentLink.Location = CurrentLink.Location + CurrentLink.Length * (FMath::Cos(IKChain.MinRotationAngleRadians) * ChildAxisX + FMath::Sin(IKChain.MinRotationAngleRadians) * ChildAxisY);
   ```

   And here is the result. 

   ![angle-limit](angle-limit.png)

## Some extra work

There is still some extra work that this anim node does for a better effect. 

### Distribute pull to re-position limb

The relevant code is: 

```cpp
		// Re-position limb to distribute pull
		const FVector PullDistributionOffset = PullDistributionAlpha * (InTargetLocation - Links[0].Location) + (1.f - PullDistributionAlpha) * (RootTargetLocation - Links.Last().Location);
		for (int32 LinkIndex = 0; LinkIndex < NumLinks; LinkIndex++)
		{
			Links[LinkIndex].Location += PullDistributionOffset;
		}
```

There is a `CVarAnimLegIKPullDistribution`(Console variable `a.AnimNode.LegIK.PullDistribution`), whose default value is `0.5f`. 

And this value means whether we care more about the foot, or the hip. 

As a result, the bone in the whole leg seems to move a delta offset. 

But `RootTargetLocation` is totally the same as `Links.Last().Location`: 

```cpp
const FVector RootTargetLocation = Links.Last().Location;
```

So what are you exactly trying to do? 

### Average pull

The relevant code is: 

```cpp
			// Pull averaging only has a visual impact when we have more than 2 bones (3 links).
			if ((NumLinks > 3) && (CVarAnimLegIKAveragePull.GetValueOnAnyThread() == 1) && (Slop > 1.f))
			{
				FIKChain ForwardPull = *this;
				FABRIK_ForwardReach(InTargetLocation, ForwardPull);

				FIKChain BackwardPull = *this;
				FABRIK_BackwardReach(RootTargetLocation, BackwardPull);

				// Average pulls
				for (int32 LinkIndex = 0; LinkIndex < NumLinks; LinkIndex++)
				{
					Links[LinkIndex].Location = 0.5f * (ForwardPull.Links[LinkIndex].Location + BackwardPull.Links[LinkIndex].Location);
				}

#if ENABLE_ANIM_DEBUG
				if (bDrawDebug)
				{
					DrawDebugIKChain(ForwardPull, FColor::Green);
					DrawDebugIKChain(BackwardPull, FColor::Blue);
				}
#endif
			}
```

To be honest, I truly have no idea how pull averaging eventually work somehow(cannot find any paper about it). Nor do I know why does pull averaging only has a visual impact when we have more than 2 bones. 

If average pull is enabled, instead of modify the original location directly, forward and backward pass would be stored separately in two `FIKChain`. The final location would be the average value of the forward and the backward value. 

## Can it be better? 

- Currently there is only a `MinRotationAngleRadians` for us to control the leg constraint. But a `MaxRotationAngleRadians` should always be concerned. 
- It would be much better if we can set the `MinRotationAngleRadians ` and `MaxRotationAngleRadians` for each joint. 







 















