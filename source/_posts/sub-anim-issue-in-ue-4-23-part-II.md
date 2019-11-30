---
title: Sub Anim Input Issue in UE 4.23, part II
mathjax: false
tags: 
 - UE4
 - Animation
 - Programming
category: UE4
date: 2019-10-18 17:19:12
description: This article briefly tells you some issues I discovered about sub anim input node in UE 4.23. This article would focus on the underlying logic that are causing this issue, compared with part I. 

---



As the previous part mentioned, 4.23 has brought some issue about the `Sub Anim Instance` node. 

## The Key Issue

The ultimate cause of this issue is about `FAnimNode_SubInput::Initialize_AnyThread`: 

```c++
void FAnimNode_SubInput::Initialize_AnyThread(const FAnimationInitializeContext& Context)
{
     if(InputProxy)
     {
          FAnimationInitializeContext InputContext(InputProxy);
          InputPose.Initialize(InputContext);
     }
}
```

The `InputPose` of the `FAnimNode_SubInput` is the link to another `Anim Graph`. 

And the old version is: 

```c++
void FAnimNode_SubInput::Initialize_AnyThread(const FAnimationInitializeContext& Context)
{
     FAnimNode_Base::Initialize_AnyThread(Context);
}
```



As a result, while `InitializeAnimation` function get called, the whole animation blueprint get initialized from the root node. 

As for initialization of the `Sub Anim Instance` node, it would traverse every node in the sub animation graph, and initialize them(of course, begin from its root node). 



**But**, instead of stopping at the `Sub Input` node, like the old version. Now the node would try to call the `Initialize_AnyThread` function of the outer graph. And that's the point. 



## Why is `StateMachine` Node Broken? 

Let us check out `FAnimInstanceProxy::InitializeRootNode` again: 

```c++
if(AnimClassInterface)
    {
        // cache any state machine descriptions we have
        for (UStructProperty* Property : AnimClassInterface->GetAnimNodeProperties())
        {

            if (Property->Struct->IsChildOf(FAnimNode_Base::StaticStruct()))
            {
                FAnimNode_Base* AnimNode = Property->ContainerPtrToValuePtr<FAnimNode_Base>(AnimInstanceObject);
                if (AnimNode)
                {
                    InitializeNode(AnimNode);

                    if (Property->Struct->IsChildOf(FAnimNode_StateMachine::StaticStruct()))
                    {
                    	FAnimNode_StateMachine* StateMachine = static_cast<FAnimNode_StateMachine*>(AnimNode);
                    	StateMachine->CacheMachineDescription(AnimClassInterface);
                    }
                }
            }
        }
    ...
```



When a ``FAnimInstanceProxy` initialize itself, it traverses its `AnimNodeProperties` and try to initialize this node. But the point is: 

**ORDER IS TRULY IMPORTANT!!! **

But how are those node properties sorted? The answer is surprisingly simple: **they are sorted by the creating order**. 

On the other hand, we came to know that a state machine description is cached after initialization. 

Imagine if you create the `Sub Anim Instance` relatively earlier than your other `State Machine` nodes. And here comes the problem:  `Sub Anim Instance` initialization may call initialize the `State Machine` node, which might not get cached yet! 



So it would be a solution to cache state machine description in advance. And in fact, this is what epic do in the future release. 



Of course, if you delete this node, and re-create a `Sub Anim Instance` node. It would work, too. Simply put the `Sub Anim Instance` node property to the tail of `AnimNodeProperties` . 



