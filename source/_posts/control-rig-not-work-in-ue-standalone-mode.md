---
title: Control Rig not Work in UE Standalone Mode 
tags: 
 - UE4
 - Control Rig
category: UE4
date: 2019-05-21 17:19:12
description: It seemes that Control Rig animation node does not work in UE standalone mode. How should I solve that? 

---

We are currently developing our own control rig system, and it works fine in editor, but it does not work in standalone mode. 

![control rig system](control rig system.png)

That is strange, and it turned out that the `LinkID` is set to `INDEX_NONE` for `Control Rig` node. 

![LinkID](Index_None.png)

And why would that happen??? 

So... We need to find where ue4 set up the `LinkID`. After some boring debug, we've found that the ue4 set the value in `FPoseLinkMappingRecord::PatchLinkIndex` function when the animation blueprint is compiled: 

![LinkID Call Stack](linkID.png)

Actually we have managed to change the `LinkID` when we click the compile button. But things just get strange when we play in standalone mode. 

Let's check the animation blueprint stuff, it seems there exists only one node in the compiled animation blueprint: 
![Blueprint Property](blueprintProperty.png)

So it is time for us to check `AnimNodeProperties` variable: 

Now we need to go through `USkeletalMeshComponent::InitAnim` function. It seems that `AnimScriptInstance` variable is already wrong, which means that `AnimClass` variable is not right(see `USkeletalMeshComponent::InitializeAnimScriptInstance` function). 
![Anim Class](AnimClass.png)

It seems that there is something wrong with module. 

---

Let's try to put the control rig graph node in develop module. Hmmmm... And it worked! 

Just place file `AnimGraphNode_Control.cpp` and `AnimGraphNode_Control.h` in the folder`ControlRigDeveloper/Private`: 

![Develop Folder](placeInDeveloper.png)

And just add `"AnimGraph"`to the PrivateDependencyModuleNames in file `ControlRigDeveloper.Build.cs`
![CS File](csFile.png)
