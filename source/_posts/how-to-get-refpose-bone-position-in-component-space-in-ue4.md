---
title: Get Ref Pose Bone Position in Component Space in UE4
tags: 
 - UE4
 - Animation
category: UE4
date: 2019-10-08 12:14:28
description: This article tells you how to get bone position of a skeleton's ref pose.   

---



Since I am engaged into something needs bone position of a ref pose in UE4, and UE4 does not provide access to it, so we need to calculate it manually. 



## If You Just Want the Answer

We can calculate bone position like this: 

```cpp
void get_ref_pose_bone_comp_space_transform(TArray<FTransform>& outResult, USkeletalMeshComponent* inSkelComp)
{
     FReferenceSkeleton refSkeleton = inSkelComp->SkeletalMesh->RefSkeleton;

     const int32 poseNum = refSkeleton.GetRefBonePose().Num(); 

     outResult.Reset(); 
     outResult.AddUninitialized(poseNum); 

     for (int32 i = 0; i < poseNum; i++)
     {
          outResult[i] = get_ref_pose_single_bone_comp_space_transform(refSkeleton, i); 

     }

}

FTransform get_ref_pose_single_bone_comp_space_transform(FReferenceSkeleton inSkel, int32 boneIdx)
{
     FTransform resultBoneTransform = inSkel.GetRefBonePose()[boneIdx];

     auto refBoneInfo = inSkel.GetRefBoneInfo(); 

     while (boneIdx)
     {
          resultBoneTransform *= inSkel.GetRefBonePose()[refBoneInfo[boneIdx].ParentIndex]; 
          boneIdx = refBoneInfo[boneIdx].ParentIndex; 
 }

     return resultBoneTransform; 
}

```

## How to Get the Bone Position of the Current Pose? 

We can use `USkeletalMeshComponent::GetComponentSpaceTransforms()` to get current bone position in component space. 

