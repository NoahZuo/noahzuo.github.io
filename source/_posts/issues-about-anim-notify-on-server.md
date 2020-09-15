---
title: Issues About Anim Notify On Server
tags: 
 - UE4
 - Animation
category: UE4
date: 2020-09-15 17:46:12
description: This article records a strange UE4.24 issue about Anim Notify that I encountered these days. 
---

# Background

So... The story begins with linked instances of our `AnimInstance`.

Everything works just fine when we have only one `AnimInstance` for our `Skeletal Mesh Component`, but it cames with the problem that only one guy can check out and edit this animation blueprint. This gradually became an issue as the team grows. 


Thus, we split our animation blueprint into several linked anim instances, which make it much easier for us to manage our anim instance. 


# The Problem Arose

It just so happens that the anim notify, which works fine when not placed in linked anim instances, are not triggered on server anymore. 

After digging into the codes, it seems that `UAnimInstance::PostUpdateAnimation()` animation is not correctly called. 

The callling stack starts from `UCharacterMovementComponent::MoveAutonomous`, which eventually leads to `USkeletalMeshComponent::TickAnimation`. 

# Linked Instance Not Handled Well
For the linked anim instance, their `PostUpdateAnimation` are called in `USkeletalMeshComponent::PostAnimEvaluation` function like this: 

```cpp
void USkeletalMeshComponent::PostAnimEvaluation(FAnimationEvaluationContext& EvaluationContext)
{
#if DO_CHECK
	checkf(!bPostEvaluatingAnimation, TEXT("PostAnimEvaluation already in progress, recursion detected for SkeletalMeshComponent [%s], AnimInstance [%s]"), *GetNameSafe(this), *GetNameSafe(EvaluationContext.AnimInstance));

	FGuardValue_Bitfield(bPostEvaluatingAnimation, true);
#endif

	SCOPE_CYCLE_COUNTER(STAT_PostAnimEvaluation);

	if (EvaluationContext.AnimInstance && EvaluationContext.AnimInstance->NeedsUpdate())
	{
		EvaluationContext.AnimInstance->PostUpdateAnimation();
	}

	for (UAnimInstance* LinkedInstance : LinkedInstances)
	{
		if (LinkedInstance && LinkedInstance->NeedsUpdate())
			LinkedInstance->PostUpdateAnimation();
	}

```

The skeletal mesh component on server, however, is always set to `Always Tick Pose`, which means that they never get evaluated. As a result, the `PostAnimEvaluation` are not called. 

We fix this issue by moving the `PostUpdateAnimation` function of linked instance into `UAnimInstance::PostUpdateAnimation()` function like this: 
```cpp
void UAnimInstance::PostUpdateAnimation()
{
#if DO_CHECK
	checkf(!bPostUpdatingAnimation, TEXT("PostUpdateAnimation already in progress, recursion detected for SkeletalMeshComponent [%s], AnimInstance [%s]"), *GetNameSafe(GetOwningComponent()), *GetName());
	TGuardValue<bool> CircularGuard(bPostUpdatingAnimation, true);
#endif

	SCOPE_CYCLE_COUNTER(STAT_PostUpdateAnimation);
	check(!IsRunningParallelEvaluation());

	if (GetSkelMeshComponent()->GetAnimInstance() == this)
	{
		for (UAnimInstance* LinkedInstance : GetSkelMeshComponent()->GetLinkedAnimInstances())
		{
			if (LinkedInstance->NeedsUpdate())
			{
				LinkedInstance->PostUpdateAnimation();
			}
		}
	}

	// Early out here if we are not set to needing an update
	if (!bNeedsUpdate)
	{
		return;
	}

```

And it worked! 

# Module Type Issue
After fixing the linked instance update issue, things are still not right. 

It works on a single process dedicated server started by editor, but not on a real server. 

It turned out that we put our customized anim nodes, `Cache Blend`, for instance, in our own plugin. And we put those `AnimGraphNodes` in an `Editor` module, which cause error like this: 

 > LogUObjectGlobals: Warning: Can't find file for asset '/Script/xxxEditor' while loading xxxxxx/AnimBP.uasset.
 > 



We should use a `Developer` or a `UncookedOnly` type for the `AnimGraphNode` module. Actually, `UncookedOnly` is the preferred type since `Developer` is deprecated. 





