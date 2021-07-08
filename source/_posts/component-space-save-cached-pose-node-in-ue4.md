---
title: Component Space Based Save&Use Cached Pose node In UE4
tags: 
 - UE4
 - Animation
category: UE4
date: 2021-06-25 11:07:03
updated: 2021-06-28 17:12:28
description: This article tells you how to implement component-space-based save&use cached pose node in UE4. 
---

# Introduction

## What Is `Save&Use Cached Pose` Node? 

UE4 has already provided a `Save Cached Pose` for one-to-many link in Anim Graph like this: 

![image-20210630114534533](image-20210630114534533.png)

We can use a `Use Save Cached Pose` node to refer a `Save Cached Pose` node. 

![image-20210630140804168](image-20210630140804168.png)

## Not-recommended Way

It actually surprised me that we can use `Reroute Node` to achieve `one to many` pose link like this: 

![image-20210630141700847](image-20210630141700847.png)

And it does work. 

However, I've talked to Epic about this issue and it turned out that this is considered as a bug. Moreover, this could cause multiple evaluation&update, which might result in some strange issue. 

In short, just don't use it. 

## What's Deep Inside ? 

Let's take a look at function `FAnimNode_SaveCachedPose::Evaluate_AnyThread`

```c++
void FAnimNode_SaveCachedPose::Evaluate_AnyThread(FPoseContext& Output)
{
	if (!EvaluationCounter.IsSynchronized_Counter(Output.AnimInstanceProxy->GetEvaluationCounter()))
	{
		EvaluationCounter.SynchronizeWith(Output.AnimInstanceProxy->GetEvaluationCounter());

		FPoseContext CachingContext(Output);
		Pose.Evaluate(CachingContext);
		CachedPose.MoveBonesFrom(CachingContext.Pose);
		CachedCurve.MoveFrom(CachingContext.Curve);
	}

	// Return the cached result
	Output.Pose.CopyBonesFrom(CachedPose);
	Output.Curve.CopyFrom(CachedCurve);
}
```

It is guaranteed to have referred nodes to be evaluated at most once per frame by synchronizing evaluation counters. And this is truly useful when building complex animation blueprints. 

# The Sadness of Space-Conversion

This method only works, however, in Local Spaces. 

This means that we need extra some space transform nodes if we want to achieve `one-to-many` pose link in component space like this: 

![image-20210630145539256](image-20210630145539256.png)

Space convert node is not that cheap since every bone is required to perform a space conversion. 

Thus we made some modification to `use cached pose` and `save cached pose` node. 

# Modification Details

## `FAnimNode_SaveCachedPose` and `UAnimGraphNode_SaveCachedPose`

Add a `FComponentSpacePoseLink` to `FAnimNode_SaveCachedPose`, and of course, a boolean property to determine if this node is under component space or local space. 

Moreover, a `FCSPose<FCompactPose>` property is also needed. Because a `FCompactPose` does not have `ComponentSpaceFlags`, which is supposed to be passed from `Save cahced pose` node to `Use cached pose` node. 

Obviously we need to override `EvaluateComponentSpace_AnyThread` function, which is simply the same as `Evaluate_AnyThread`. 

```cpp
void FAnimNode_SaveCachedPose::EvaluateComponentSpace_AnyThread(FComponentSpacePoseContext& Output)
{
	if (!EvaluationCounter.IsSynchronized_Counter(Output.AnimInstanceProxy->GetEvaluationCounter()))
	{
		EvaluationCounter.SynchronizeWith(Output.AnimInstanceProxy->GetEvaluationCounter());

		FComponentSpacePoseContext CachingContext(Output);
		ComponentPose.EvaluateComponentSpace(CachingContext);
		
		CachedComponentPose = MoveTemp(CachingContext.Pose);
		CachedCurve.MoveFrom(CachingContext.Curve);
	}

	// Return the cached result
	Output.Pose.CopyPose(CachedComponentPose);
	Output.Curve.CopyFrom(CachedCurve);
}
```

Of course, don't forget to handle `CacheBones_AnyThread`, `Initialize_AnyThread` and `PostGraphUpdate` function. 

As for `UAnimGraphNode_SaveCachedPose`, we need to override its `CustomizePinData` and `PostEditChangeProperty` function: 

```cpp
void UAnimGraphNode_SaveCachedPose::CustomizePinData(UEdGraphPin* Pin, FName SourcePropertyName, int32 ArrayIndex) const
{

	//CreatePin(EGPD_Output, UAnimationGraphSchema::PC_Struct, FPoseLink::StaticStruct(), TEXT("Pose"));
	Super::CustomizePinData(Pin, SourcePropertyName, ArrayIndex);

	if (Pin->PinName == GET_MEMBER_NAME_STRING_CHECKED(FAnimNode_SaveCachedPose, Pose))
	{
		Pin->bHidden = (Node.UseComponentSpace);
		//Pin->Direction = Node.UseComponentSpace ? EGPD_MAX : EGPD_Input;
	}
	if (Pin->PinName == GET_MEMBER_NAME_STRING_CHECKED(FAnimNode_SaveCachedPose, ComponentPose))
	{
		Pin->bHidden = (!Node.UseComponentSpace);
		//Pin->Direction = (!Node.UseComponentSpace) ? EGPD_MAX : EGPD_Input;
	}
}

void UAnimGraphNode_SaveCachedPose::PostEditChangeProperty(struct FPropertyChangedEvent& PropertyChangedEvent)
{
	const FName PropertyName = (PropertyChangedEvent.Property ? PropertyChangedEvent.Property->GetFName() : NAME_None);

	// Reconstruct node to show updates to PinFriendlyNames.
	if (PropertyName == GET_MEMBER_NAME_STRING_CHECKED(FAnimNode_SaveCachedPose, UseComponentSpace))
	{
		ReconstructNode();
		//FBlueprintEditorUtils::MarkBlueprintAsStructurallyModified(GetBlueprint());
	}

	Super::PostEditChangeProperty(PropertyChangedEvent);
}
```

## `FAnimNode_UseCachedPose` and `UAnimGraphNode_UseCachedPose`

First of all, we should add a `FComponentSpacePoseLink` property, just like `LinkToCachingNode`. 

Then override `EvaluateComponentSpace_AnyThread` function that simply evaluate the component space pose link. 

Still, we need to override `CreateOutputPins` function to eventually create component space out pins. 

```cpp
void UAnimGraphNode_UseCachedPose::CreateOutputPins()
{
	//if UseComponentSpace Create Output Icon By ComponentSpacePose Link
	//else Create Default Output Icon
	if (SaveCachedPoseNode.IsValid() && SaveCachedPoseNode->Node.UseComponentSpace)
	{
		CreatePin(EGPD_Output, UAnimationGraphSchema::PC_Struct, FComponentSpacePoseLink::StaticStruct(), TEXT("Pose"));
	}
	else
	{
		Super::CreateOutputPins();
	}
}
```

## Link Pose Together

We need to link `FAnimNode_UseCachedPose::LinkToCachingNode_Component` to  `FAnimNode_SaveCachedPose::ComponentPose`. 

To achieve this, we should modify `FAnimBlueprintCompilerContext::ProcessUseCachedPose` function

```cpp
if (UAnimGraphNode_SaveCachedPose* AssociatedSaveNode = SaveCachedPoseNodes.FindRef(UseCachedPose->SaveCachedPoseNode->CacheName))
{
	UseCachedPose->Node.UseComponentSpace = AssociatedSaveNode->Node.UseComponentSpace;
	FStructProperty* LinkProperty = FindFProperty<FStructProperty>(FAnimNode_UseCachedPose::StaticStruct(), AssociatedSaveNode->Node.UseComponentSpace?TEXT("LinkToCachingNode_Component"):TEXT("LinkToCachingNode"));
	check(LinkProperty);
 ... 
```

Last but not least, we should handle such warning: 

![image-20210708165338511](image-20210708165338511.png)

In order to solve this issue, we should modify `FAnimBlueprintCompilerContext::ProcessAnimationNode` function: 

```cpp
		if (!bConsumed && (SourcePin->Direction == EGPD_Input))
		{
			//For SaveCachedPose, a hidden input needs to be ignored  
			if (!SourcePin->bHidden)
			{
				//@TODO: ANIMREFACTOR: It's probably OK to have certain pins ignored eventually, but this is very helpful during development
				MessageLog.Note(TEXT("@@ was visible but ignored"), SourcePin);
			}
		}
```

And that's all! Everything is good to go: 

![enter image description here](tapd_20418141_1625033370_95.png)

Enjoy this feature. 

