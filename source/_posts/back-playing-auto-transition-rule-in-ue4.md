---
title: Automatic Rule Based Transition For Back-playing Animation
date: 2020-07-14 12:43:56
tags:
 - UE4
 - Animation
category: UE4
description: This article simply tells you how to implement automatic rule based transition for a back-playing animation, which is used in a state machine in UE4. 
---

# What is `Automatic Rule Based Transition`? 
`Automatic rule based transition` is used, in most cases, if you want an auto-transition from source state to target state when the source animation ends. 

For example, consider a state transition from State `Jump_Land` to `Idle`: 

![jump_land_to_idle](jump_land_to_idle.png)

An alternative way to implement state transition condition is to use `Get Relevant Time Remaining` node like this: 

![yet-another-method](yet-another-method.png)

But this is, to some extent, dirty and expensive. We cannot use `Animation` variable with it, even worse, `Fast Path` does not work with this way. 

---

The proper method in this case is to check `Automatic Rule Based on SequencePlayer in State` of this transition: 

![arbt](arbt.png)

Transition would start when there is 0.2 second left in the source state animation. This is clean and elegant. 

# Problem With Back-Playing Sequence

For the official UE4 engine, `Automatic Rule Based Transition` does not work with a back-playing sequence. Because transition only cares about the accumulated time of an animation asset. 

```c++
if (UAnimationAsset* AnimAsset = RelevantPlayer->GetAnimAsset())
{
	const float AnimTimeRemaining = AnimAsset->GetMaxCurrentTime() - RelevantPlayer->GetAccumulatedTime();
	const FAnimationTransitionBetweenStates& TransitionInfo = GetTransitionInfo(TransitionRule.TransitionIndex);
	bCanEnterTransition = (AnimTimeRemaining <= TransitionInfo.CrossfadeDuration);
}
```

As a result, for a back-playing animation sequence, it does not work as expected. Hence some modifications are needed: 

```c++
// FAnimNode_StateMachine::FindValidTransition
if (UAnimationAsset* AnimAsset = RelevantPlayer->GetAnimAsset())
{
	// Noah Zuo: Here we need to make it work with an asset player of minus play rate. 
	const bool AnimReversePlay = RelevantPlayer->GetEffectivePlayRate() < 0.f;
	const float AnimTimeRemaining = AnimReversePlay ? RelevantPlayer->GetAccumulatedTime() : AnimAsset->GetMaxCurrentTime() - RelevantPlayer->GetAccumulatedTime();

	const FAnimationTransitionBetweenStates& TransitionInfo = GetTransitionInfo(TransitionRule.TransitionIndex);
	// For a back-running playing asset player, we need to flip condition. 
	bCanEnterTransition = (AnimTimeRemaining <= TransitionInfo.CrossfadeDuration);
}
```

We need to add a function for each `FAnimNode_AssetPlayerBase` to get real play rate: 

```cpp
// FAnimNode_AssetPlayerBase
virtual float GetEffectivePlayRate() { return 1.f; }
```

And override this function in `FAnimNode_SequencePlayer`: 

```c++
float FAnimNode_SequencePlayer::GetEffectivePlayRate()
{
	const float SequencePlayRate = (Sequence ? Sequence->RateScale : 1.f);
	const float AdjustedPlayRate = PlayRateScaleBiasClamp.ApplyTo(FMath::IsNearlyZero(PlayRateBasis) ? 0.f : (PlayRate / PlayRateBasis), 0.f);
	const float EffectivePlayrate = SequencePlayRate * AdjustedPlayRate;
	return EffectivePlayrate; 
}

```

And that's it! Enjoy this feature! 







 

