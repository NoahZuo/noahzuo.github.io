---
title: PhysX Scratch Buffer in Unity 
tags: 
 - Game Physics
 - Programming
 - Unity
 - physX
category: Unity
date: 2019-07-17 20:24:39
description: This article tells you something about physX scratch buffer in Unity. 


---



## What is Scratch Buffer in PhysX? 

We always encounter bottleneck in `scene::simulate` function. When there are too many rigid bodies or joints in the scene, a lot of performance is required to complete those simulations. 



> As far as possible, PhysX will internally allocate temporary buffers 
> from the scratch memory block, thereby reducing the need to perform 
> temporary allocations from PxAllocatorCallback. 



So... In the function`scene::simulate`, scratch buffer is pushed into the scratch allocator. And this means that physX would try to allocate temp buffers inside the scratch buffer. 



It needs to be concerned that: 



> Much of the memory PhysX uses for simulation is held in a pool of blocks, each 16K in size.



Thus we need to make sure that the buffer size should be multiple of 16k. And in addition, this buffer should be 16-byte aligned. 



We can use `uint8` and `malloc` to do this. 



## Scratch Buffer in Unity

Surprisingly, Unity did nothing about scratch buffer(Well... at least for 5.6). It simply passes fixed delta time to `physicsScene::simulate` function. And this means that it does not use scratch buffer at all. 



As a result, `Physics.simulate` costs too much performance in our rag doll scene. Especially for those low-end device. 



And it is time for us to modify the engine code. 



## Our Modification

We created an int variable in physics manager class. This value means the block number of scratch buffer we need. The default value is 0, which means we still do not use scratch buffer. 



![Scratch Buffer](scratch buffer.png)



We created the scratch buffer in the physics manager constructor function. And destroy this buffer in the destructor function. 



Moreover, we add some terminal function to re-allocate scratch buffer during runtime. This makes us easily to compare performance cost between different set of scratch buffer. 



And the result is inspiring. Scratch buffer saved us lots of performance for our regular scene. 