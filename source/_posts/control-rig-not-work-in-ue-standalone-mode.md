---
title: Control Rig not Work in UE Standalone Mode 
tags: 
 - UE4
 - Control Rig
category: UE4
description: It seemes that Control Rig animation node does not work in UE standalone mode. How should I solve that? 

---

We currently customized our own control rig system, and it works fine in editor, but it does not work in standalone mode. 

[control rig system](control rig system.png)

That is strange, and it turned out that the `LinkID` is set to `INDEX_NONE` for `Control Rig` node. 
[LinkID](Index_None.png)

And why would that happen??? 

So... We need to find where ue4 set up the `LinkID`. After some boring debug, we've found that the ue4 set the value in `FPoseLinkMappingRecord::PatchLinkIndex` function when the animation blueprint is compiled: : 
[LinkID Call Stack][linkID.png]

Actually we have managed to change the `LinkID` when we click the compile button. But things just get strange when we play in standalone mode. 

Let's check the animation blueprint stuff, it seems there exists only one node in the compiled animation blueprint: 
[Blueprint Property](blueprintProperty.png)

So it is time for us to check `AnimNodeProperties` variable: 

Now we need to go through `USkeletalMeshComponent::InitAnim` function. It seems that `AnimScriptInstance` variable is already wrong, which means that `AnimClass` variable is not right(see `USkeletalMeshComponent::InitializeAnimScriptInstance` function). 
[Anim Class](AnimClass.png)
Strange... Still working on it. 


