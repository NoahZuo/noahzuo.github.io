---
title: Extending Two-bone IK for Left Hand Holding Guns
mathjax: true
tags: 
 - UE4
 - Animation
category: UE4
date: 2020-12-11 10:02:51
updated: 2020-12-13 15:27:03
description: This article tells you how I extend two-bone IK in UE4 to hold guns properly. 
---



# Introduction

`Two bone IK` is always used for arm IK solve. That is to say, the triangle `Hand-Elbow-Arm` is solved using  [Law of Cosines](https://en.wikipedia.org/wiki/Law_of_cosines). 



The hand bone of our character is always located, however, at position of the wrist of character mesh like this: 

![image-20210103200637671](image-20210103200637671.png)

But we don't directly place this bone to the `Left Hand Target` socket of the weapon mesh, or else: 

![image-20210103202118602](image-20210103202118602.png)

The palm bone of left hand should be constrainted to `Left Hand Target` of the weapon mesh. So, we need to solve `Arm-Elbow-Hand-Palm` chain. 

As a result, default `Two Bone IK` node cannot meet our requirement. We need our own anim node. 

# Why doesn't `Two bone IK` work? 

We need a valid location of `Left Hand Bone` calculated by: 
$$
M_{LeftHand} = M_{Effector}* M_{LeftPalm}^{-1}
$$

which `$M_Effector$` is the matrix of palm target, and $M_{LeftPalm}^{-1}$ is the inverse of the local transform of `Left Palm` bone. 



Of course we can calculate $M_{LeftPalm}^{-1}$ in the `Event Graph` or somewhere else. But this is the local transform from previous frame, which might  be different of that in current frame. 

In short, we need to get this inverse matrix when this `AnimInstance` get evaluated. To achieve that, I simply created a new anim node. 

# Key Logic

The implementation is truly easy because we just need to apply a inversed transform to the `Effector` before a `AnimationCore::SolveTwoBoneIK` is called. 

Thus we just need to get the socket or bone local transform, and apply its inversed value to the current `EffectorTransform`. 

```cpp
	EffectorTransform =  holdingPointLocalTransform.Inverse() * EffectorTransform;	

	// IK solver
	UpperLimbCSTransform.SetLocation(RootPos);
	LowerLimbCSTransform.SetLocation(InitialJointPos);
	EndBoneCSTransform.SetLocation(InitialEndPos);

	AnimationCore::SolveTwoBoneIK(UpperLimbCSTransform, LowerLimbCSTransform, EndBoneCSTransform, JointTargetPos, DesiredPos, bAllowStretching, StartStretchRatio, MaxStretchScale);

```

We can set the palm bone or socket for this node like this: 

![image-20210120173535279](image-20210120173535279.png)

And this is what we eventually have: 

![result](result.gif)