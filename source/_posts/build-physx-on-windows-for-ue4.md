---
title: Build PhysX For UE4 On Windows
tags: 
 - UE4
 - Physics
category: UE4
date: 2021-08-31 11:28:12
updated: 2021-11-16 17:34:28
description: This article records how to build PhysX binaries for UE4. 
---



# Introduction

Sometimes we need to debugging PhysX for UE4. But it seems `.pdb` files are not provided when it comes to PhysX. 

And sometimes we might want to modify PhysX code, thus we need to build our own PhysX binaries. 

You need Visual Studio 2015 and CMake installed on your computer. Make sure you have C++ installed for your VS 2015. 

# Steps 1: Generate Visual Studio Files

This step is fairly easy--simply lanch CMake, fill `Source Code Location` and `Binary Location`: 

![image_00](image_00.png)

Hit `Configure`, select `Win64` as platform, and wait...

![image_01](image_01.png)

Oops! one error, if my guess is right, would pop up during configuration stage: 

![image_02](image_02.png)

It seems that `GW_DEPS_ROOT` is not set. Click `Enviroment`, and `Add Entry` like this: 

![image_03](image_03.png)

This should work! Click `Configure` again, and then `Generate`. 

Now we have our Visual Studio files. 

![image_04](image_04.png)

# Step 2: Build PhysX Project

I truly believe that there is no need to talk about... compiling the project. 

You can compile any mode of PhysX you like. I prefer `profile` mode personly. 

It should be noted that different mode of PhysX would be loaded in accordance with different engine mode. You can check out `GetPhysXLibraryMode` function in `PhysX.Build.cs` for details. 

![image_05](image_05.png)

After building PhysX, you can find dlls and pdbs: 

![image_06](image_06.png)

# Step 3: Let's Party! 

Simply copy our dlls and pdbs to the binary folder in our engine. 

And now we can dive into PhysX's underlying logic! 

![image_07](image_07.png)

If you still cannot dive into PhysX, check your module: 

![image_08](image_08.png)

And load PhysX module manually: 

![image_09](image_09.png)



That's all! And enjoy it! 