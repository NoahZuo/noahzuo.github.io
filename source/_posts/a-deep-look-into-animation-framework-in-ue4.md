---
title: A deep look into animation framework in ue4
tags: 
 - UE4
 - Animation
 - Programming
category: UE4
date: 2019-09-11 13:40:12
description: This article introduces the animation framework in ue4, in a programmer's perspective. 
---


It seems that Epic prefers telling developers how to use animation blueprint in Unreal Engine 4. But me, as a programmer, still needs to get to know what the animation system actually works underlying. 



## Relevant Data Structure



### `USkeletalMeshComponent`

This is the where we setup our skeletal mesh in UE4. We use this component to handle `game-play` stuff. But it seems that UE4 want to decoupling game-play and animation system, as a result, you cannot write bone transform using this component. 



### `UAnimInstance`

You may not be very familiar with this class. But you must be very familiar with its child classes -- `AnimBlueprint`. 

We can choose an `Anim Class` for a `USkeletalMeshComponent`. As we just mentioned that UE4 want to decoupling game-play and animation system. And since `USkeletalMeshComponent` focus on game-play stuff, it needs `UAnimInstance` to handle animation stuff. 

```c++
UAnimInstance* USkeletalMeshComponent::GetAnimInstance() const
{
    return AnimScriptInstance;
}
```

`AnimBlueprint` always inherits from `UAnimInstance`. I am pretty sure that you have played with `Blueprint Update Animation` event... We always setup our animation data in this event. 

![Update-Event](update-event.png)



This event is called in `UAnimInstance::UpdateAnimation()` function, which is called in `USkeletalMeshComponent::TickPose()`, which is called in `USkeletalMeshComponent::TickComponent`. 



By going through `UAnimInstance::UpdateAnimation` function, it's easy to know what is happening in an `AnimBlueprint`.  

1. By `PreUpdateAnimation`, the `UAnimInstance` would call `PreUpdate` function of its `AnimInstanceProxy`. And an `AnimNode` uses `PreUpdate` to update its game-play data. 
2. Then montages would be updated. Montage needs to be updated BEFORE node update or Native Update, so that node knows where montage is. 
3. Next it is time to gather game-play data for the `AnimInstance`. For C++ stuff, `NativeUpdateAnimation` is called. For Blueprint, `BlueprintUpdateAnimation` is called. And this is exactly `Update Event` in Anim Blueprint. 
4. Then `PostUpdateAnimation` is called. At this point, notifies are handled. 



Seems simple, isn't it? 

### `FAnimInstanceProxy`

`FAnimInstanceProxy` is : 

> the proxy object passed around during animation tree update in lieu of a `UAnimInstance`. 

This class contains almost everything you need for your `AnimGraph`. You can handle bones, notifies, state machines, pose snapshot, etc, in this struct. 

`FAnimInstanceProxy` has the access to the root node of the `AnimGraph`. UE4 does a recursive update or evaluation starting from the root node. 















