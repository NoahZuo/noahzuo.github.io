---
title: Sub Anim Input Issue in UE 4.23, part I
mathjax: false
tags: 
 - UE4
 - Animation
date: 2019-09-17 17:40:12
category: UE4
description: This article briefly tells you some issues I discovered about sub anim input node in UE 4.23. 

---



Some issues about animation blueprint have just been discovered when I try to upgrade my 4.20 project to 4.23. And it turned out that those issues had caused some trouble to us. 

As a result, it took me several days to dig into this issue, and something seems should be recorded. Hope this could help you some day. 



## What Is This Issue? 

When you try to open an `Animation Blueprint` **with a `Sub Anim Instance` node in it**, you may notice that the engine is frozen. And while you check the editor log, you see lots of: 

>  FAnimNode_StateMachine: Bad machine ptr 



And you may notice that the engine takes much more time to compile the `Animation Blueprint` that have a `Sub Anim Instance` node in it. 



Now you may get the point: This issue is all about the `Sub Anim Instance` node. 



## If You Simply Want a Solution

### Method 1: Modify the engine code

By modify the engine code, you can fix the issue once and for all. 

Modify the  `FAnimInstanceProxy::InitializeRootNode`  function in the `AnimInstanceProxy.cpp` file. 

You need to move 

```c++
        for (UStructProperty* Property : AnimClassInterface->GetAnimNodeProperties())
        {
            if (Property->Struct->IsChildOf(FAnimNode_StateMachine::StaticStruct()))
            {
                FAnimNode_StateMachine* StateMachine = Property->ContainerPtrToValuePtr<FAnimNode_StateMachine>(AnimInstanceObject);
                StateMachine->CacheMachineDescription(AnimClassInterface);
            }
        }
```

to the top of the `if` block. 

The final code looks like: 

```c++
if(AnimClassInterface)
    {
        // cache any state machine descriptions we have
        for (UStructProperty* Property : AnimClassInterface->GetAnimNodeProperties())
        {
            if (Property->Struct->IsChildOf(FAnimNode_StateMachine::StaticStruct()))
            {
                FAnimNode_StateMachine* StateMachine = Property->ContainerPtrToValuePtr<FAnimNode_StateMachine>(AnimInstanceObject);
                StateMachine->CacheMachineDescription(AnimClassInterface);
            }
        }

        // cache any state machine descriptions we have
        for (UStructProperty* Property : AnimClassInterface->GetAnimNodeProperties())
        {

            if (Property->Struct->IsChildOf(FAnimNode_Base::StaticStruct()))
            {
                FAnimNode_Base* AnimNode = Property->ContainerPtrToValuePtr<FAnimNode_Base>(AnimInstanceObject);
                if (AnimNode)
                {
                    InitializeNode(AnimNode);

                    //if (Property->Struct->IsChildOf(FAnimNode_StateMachine::StaticStruct()))
                    //{
                    //	FAnimNode_StateMachine* StateMachine = static_cast<FAnimNode_StateMachine*>(AnimNode);
                    //	StateMachine->CacheMachineDescription(AnimClassInterface);
                    //}
                }
            }
        }

        // Cache default sub-input 
        for(const FAnimBlueprintFunction& AnimBlueprintFunction : AnimClassInterface->GetAnimBlueprintFunctions())
        {
            if(AnimBlueprintFunction.Name == NAME_AnimGraph)
            {
                check(AnimBlueprintFunction.InputPoseNames.Num() == AnimBlueprintFunction.InputPoseNodeProperties.Num());
                for(int32 InputIndex = 0; InputIndex < AnimBlueprintFunction.InputPoseNames.Num(); ++InputIndex)
                {
                    if(AnimBlueprintFunction.InputPoseNames[InputIndex] == FAnimNode_SubInput::DefaultInputPoseName && AnimBlueprintFunction.InputPoseNodeProperties[InputIndex] != nullptr)
                    {
                        DefaultSubInstanceInputNode = AnimBlueprintFunction.InputPoseNodeProperties[InputIndex]->ContainerPtrToValuePtr<FAnimNode_SubInput>(CastChecked<UAnimInstance>(GetAnimInstanceObject()));
                        break;
                    }
                }
            }
        }
    }
    else
    {
        //We have a custom root node, so get the associated nodes and initialize them
        TArray<FAnimNode_Base*> CustomNodes;
        GetCustomNodes(CustomNodes);
        for(FAnimNode_Base* Node : CustomNodes)
        {
            if(Node)
            {
                InitializeNode(Node);
            }
        }
    }
```



### Method 2: Modify the Resource 

If you tend to keep consistent with the official engine, you need to follow these steps: 

1. Add following lines to your `DefaultEngine.ini` file. 

```ini
[Core.Log]
LogAnimation=Error
```

This would prevent the engine from printing endless log to your console, which causing freeze. 

2. Open the Animation Blueprint, and locate the `Sub Anim Input` node. This might take some time... 
3. Create a new `Sub Anim Input` node and replace the old node with the new one. 
4. Revert those modifications from the `DefaultEngine.ini` file. 



So that's how we should fix this issue. Another blog would be post to record how I dig into this issue. 



