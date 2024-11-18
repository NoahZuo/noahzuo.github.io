---
title: Crash When Opening a Control Rig File in UE4
tags: 
 - UE4
 - Animation
 - Control Rig
category: UE4
date: 2019-05-21 21:29:29
description: This article introduced how I solve a crash problem when open control rig file in Unreal Engine 4. 

---



If by any chance that you encounter a crash when open a control rig file, this article might help you. 

---

## How I met this problem? 

Actually... I do have no idea in the very beginning. Since I am doing some customization and have made lots of modification to the plugin code. Actually it worked fine and crash occurred in the next day. 

And the crash is like this: 

![Crash](crash.png)

![Call Stack](crash call stack.png)

## What is this problem? 

It seems that `FControlRigEitor::GetAssetEditorModeManager()` function returns a nullptr? Which means there is something wrong with the editor. 

Now let's see where function `SetAssetEditorModeManager` is called. According to the code comments, control rig editor is much the same as Animation Blueprint Editor, so I'll check anim bp editor source code. 

I finally found call stack like this: 

![Anim BP Call Stack](control rig editor.png)

So... Yes! This should be something wrong about `persona` viewport. 

## How to solve this problem? 

Firstly, I'll add an `if... else` condition to check if function `GetAssetEditorModeManager()` returns null to avoid crash. 

![Add a guard](add guard.png)

And control rig file was opened successfully. Only with viewport closed. 

![Open file](open file.png)

Oh... I see! The problem is that I closed the viewport before exit UE4, and this result in function `GetAssetEditorModeManager()` returning null. 

To prove this, I re-opened the viewport, and exit UE4, then revert my modification, compile, yep!!! It worked! 

## Final result
So the final result, is to add a guard to function `GetAssetEditorModeManager()` to avoid crash. 

And remember to add a guard to `GetEditMode()` because this function would call function `GetAssetEditorModeManager()` without check. 
![GetEditMode](geteditmode.png)

Hope this article could help you... 