---
title: A Small Issue about Unity API Updater
date: 2024-11-24 15:02:14
updated: 2024-11-29 20:37:27
tags: 
 - Unity
category: Unity
description: This article discusses a Unity plugin issue when adapting a DLL to different versions and providing a solution using command-line parameters for API updates.
---

# Introduction
In Unity development, writing code in C# and then compiling it into a DLL for distribution is a common practice for creating plugins. However, if you want a DLL to be compatible with different versions of Unity, things become more complicated.

This article documents the issues I encountered during the process of building such a DLL and the eventual solution.

# What is the Problem?
Our DLL relies on Unity 2021 DLLs and performs a `Memory Take Snapshot` operation, as shown in the following code:

```cs
UnityEngine.Profiling.Memory.Experimental.MemoryProfiler.TakeSnapshot(targetPath, postTakeSnapshotCallback); 
```

When this DLL is used in a Unity 2021 project, everything works fine. However, when the DLL is used in a Unity 2022 project, Unity prompts the following:

 > These precompiled assemblied: 
 > Packages/xxx/xxx.dll use Unity API's that have changed since these assemblies were built. Unity's API updater can modify these precompiled assemblies to adjust them to the changed Unity API so they can funtion correctly again. Not letting them be updated willl cause them to likely fail at run or load time. 

![Consent Request](ConsentRequest.png)

After clicking the `Yes` button, this DLL will be modified by Unity to adapt to the Unity 2022 engine, and everything will work correctly.

However, for those building machines in the project team, it is common to forget to click the `Yes` button. Moreover, build machines usually start in `batch mode`, leaving no opportunity to click this button. As a result, the DLL is incompatible with the engine version, leading to C++ compilation errors during the build.

# Solution
Unity's official documentation mentions that when starting in `batch mode`, you can add the -accept-apiupdate parameter to the command, which will automatically perform the API update upon startup.

In addition, there is an undocumented command-line parameter: `-skipUpgradeDialogs`, which serves the same purpose. This parameter also works when starting the editor, automatically performing the API update.
