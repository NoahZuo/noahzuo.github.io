---
title: Get Access to SkeletalMesh Vertices in UE4
tags: 
 - UE4
 - Animation
 - Skinning
category: UE4
date: 2019-09-13 17:50:33
description: This article tells you how to get access to skeletalmesh vertices in UE4.   

---


## For GPU Skinned Skeletal Mesh

UE4 use GPU skin for skeletal mesh by default. In that case position of those skeletal mesh vertices are calculated on GPU. So we can only manually calculate the final position on CPU side again. And this would very much likely to cause a performance issue. 

There is a helper function`USkeletalMeshComponent::ComputeSkinnedPositions()` to get exact the vertex position. It seems to be the only way to manually get the vertice position. 

## For CPU Skinned Skeletal Mesh

It would be much cheaper to get vertice position for a cpu skinned skeletal mesh. Vertice position would be cached after skinning to a skeleton. 

We can directly use the cached positions to save performance like this: 

```c++
static_cast<FSkeletalMeshObjectCPUSkin*>(MeshObject)->GetCachedFinalVertices(); 
```

There is a `FSkeletalMeshObject` called `MeshObject` in a `USkinnedMeshCompnent`. This pointer can point to a `FSkeletalMeshObjectCPUSkin` or a `FSkeletalMeshObjectGPUSkin`.  This object is responsible for sending stuffs like bone transforms, morph target state to render thread. 

Since a cpu skinned skeletal mesh caches its vertices every time it updates dynamic data(See `FSkeletalMeshObjectCPUSkin::UpdateDynamicData_RenderThread()` and `FSkeletalMeshObjectCPUSkin::Update()`). The final vertice position is stored and thus we have access to it. 

## Enable CPU Skinning

UE4 use cpu skinning only if the feature level is not valid or the skeletal mesh got cloth to simulate. We need to explicitly set `bCPUSkinning` to true if we want to enable cpu skinning manually. 


But it is not enough only to set `bCPUSkinning` to true. We still need to refresh render data to make it work. 


An easy but expensive method would be recreating renderstate and then flushing render commands after changing this variable like this:  

```c++
bCPUSkinning = true; 
RecreateRenderState_Concurrent(); 
FlushRenderingCommands(); 
```
