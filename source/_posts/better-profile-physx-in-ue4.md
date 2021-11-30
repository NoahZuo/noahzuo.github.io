---
title: Better Way To Profile PhysX In UE4
tags: 
 - UE4
 - PhysX
 - Physics
category: UE4
date: 2021-11-27 11:15:12
updated: 2021-11-30 16:21:28
description: This article records how to streaming pvd data to a file for better PhysX Profile. 
---



# Introduction

It still remains hard to profile PhysX when you are developing a UE4 project. Of course you can use `Heat Maps(For Linux)` or `StreamLines(for Android)` to get very in-depth performance data like: 

![image_1](image_1.png)

This tools can really help a lot to find performance hotspots. But pvd should be used if you want to digging into PhysX since  measuring scope performance is pretty common during performance profile. 



# How Does UE4 Handle PVD? 

UE4 handles PVD using `GPhysXVisualDebugger`, which is created in `InitGamePhysCore` function. You need to call `PvdConnect` function by console command to connect to `PhysX Visual Debugger` running on your local machine like: 

```cpp
pvd connect localhost
```

But there is a huge limitation with this method--You cannot easily collect PVD data on dedicated server or mobile device. 



# Streaming To File

Obviously, it would be much better to streaming pvd data to a file, which can be loaded later in `PVD`. 

## Global Parameters Setup

Instead of simply creating two functions to handle connection, we choose to create two Global Parameters, `GConnecttoPVD`  and `GDisconnectFromPVD` since we want to handle pvd connection in `FPhysScene_PhysX::StartFrame()`. 



You may also handle connection in other stages as well. But I still recommend `FPhysScene_PhysX::StartFrame()` because `PVD` is known as easy to crash. 



## Connect&Disconnect

And there is a hint: We might want to name our file by date like: 

```cpp
FString TextToSave = FDateTime::Now().ToString();
TextToSave.Append(".pxd2");
```

So, we can connect to `PVD` file like: 

```cpp
PxPvdTransport* transport = PxDefaultPvdFileTransportCreate(TCHAR_TO_ANSI(*TextToSave));
GPhysXVisualDebugger->connect(*transport, PxPvdInstrumentationFlag::eALL );
```

Here, `PxPvdInstrumentationFlag::eALL` means streaming everything, which includes profile, memory and even all visualized debug data, to your file. Hence this cause a very large file, and it is very easy to crash during loading this file. 

Thus, I personally recommend to specify `PxPvdInstrumentationFlag::ePROFILE` only when creating PVD file. 



And as for disconnecting from PVD, we simply call: 

```cpp
GPhysXVisualDebugger->disconnect();
```

And it's done! 

# Recommended Way To Connect&Disconnect

Personally, I love to specify two timers to manually handle `PVD` connections by `FParse::Value(FCommandLine::Get(), TEXT("-PVDTimer="), Value)`. 

Try to avoid connecting to `PVD` at start to prevent crash. Sigh... 