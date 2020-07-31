---
title: Use SaveCachedPose inside State Machine in UE4 
tags: 
 - UE4
 - Animation
category: UE4
date: 2020-07-30 16:31:12
updated: 2020-07-30 16:31:12
description: This article tells you how to use SaveCachedPose node inside a state machine by modifying engine source code. 

---



# What is a `SaveCachedPose` node? 

We can always use `SaveCachedPose` node to cache the output pose, which can be used in the main tree, to avoid endless copy&paste of anim graph in an `Animation Blueprint`. 

![savecachedpose](savecachedpose.png)

It is this node that can reduce complexity of an `Anim Graph`, except you cannot use it inside a state machine. 

# Digging into code

It seems that using a `Save Cached Pose` node inside a state machine requires a lot of work to grab. I've seen comment in the code like this: 

```cpp
		else if (UAnimGraphNode_SaveCachedPose* SavePoseRoot = Cast<UAnimGraphNode_SaveCachedPose>(SourceNode))
		{
			//@TODO: Ideally we only add these if there is a UseCachedPose node referencing them, but those can be anywhere and are hard to grab
			SaveCachedPoseNodes.Add(SavePoseRoot->CacheName, SavePoseRoot);
			RootSet.Add(SavePoseRoot);
		}
```

But frankly speaking it is not hard at all, for animation systems that have been refactored over and over again. We only need to modify serveral lines of code to implement this feature. 



# How to implement

## Enable `SaveCachedPose` node inside a state machine

First of all, we should make `SaveCachedPose` appear in the state machine node list. Thus, we should modify `UAnimGraphNode_SaveCachedPose::IsCompatibleWithGraph` function. 

This function is like: 

```cpp
	bool const bIsNotStateMachine = TargetGraph->GetOuter()->IsA(UAnimBlueprint::StaticClass());
	return bIsNotStateMachine && Super::IsCompatibleWithGraph(TargetGraph);
```

We simply make variable `bIsNotStateMachine` always true: 

```cpp
	bool const bIsNotStateMachine = true;
	return bIsNotStateMachine && Super::IsCompatibleWithGraph(TargetGraph);
```

As a result, we can add a `SaveCachedPose` node inside a state machine. 

## Fix compile issue of the `Animation Blueprint`

Things cannot simply go so well. Some compile error arose when we actually use this node inside a state machine like this: 

![Compile Errors](compile-errors.png)

### The first compile error

After digging into the call stack, it seems that something is wrong in `ExpandGraphAndProcessNodes` function: 

```cpp
	// Process any animation nodes
	{
		TArray<UAnimGraphNode_Base*> RootSet;
		RootSet.Add(TargetRootNode);
		ProcessAnimationNodesGivenRoot(AnimNodeList, RootSet);
	}
```

As a `SaveCachedPose` node shall also be viewed as a so-called *root*. Thus we need to add these node to the `RootSet` as well like this: 

```cpp
	// Process any animation nodes
	{
		TArray<UAnimGraphNode_Base*> RootSet;
		RootSet.Add(TargetRootNode);
		RootSet.Append(SaveCachedPoseNodesInStateGraph);
		ProcessAnimationNodesGivenRoot(AnimNodeList, RootSet);
	}
```

And it seems that the first compile error just gone. And we still have one more to deal with. 

![second error](second-error.png)

### The second compile error

According to the call stack, it turned out that the `SaveCachedPoseNodes` does not contain a node that our `UseCachedPose` node referred. 

![second-error-reason](second-error-reason.png)

Still very easy to deal with. We just need to add this `SaveCachedPose` node to `FAnimBlueprintCompilerContext::SaveCachedPoseNodes` in `ExpandGraphAndProcessNodes` function. 

And it just works! 











 

