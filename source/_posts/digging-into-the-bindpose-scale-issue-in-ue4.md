---
title: Digging Into The Bind Pose Scale Issue In UE4
tags: 
 - UE4
 - Animation
category: UE4
date: 2022-04-12 10:17:51
updated: 2022-04-12 10:17:51
description: This article records an issue about bone scale in bind pose that I've encountered these days. 
---

# What's The Issue?

For a special, or rather, a stupid reason, sometimes the bone scale in a bind pose can have a non-identity value. Let me address this issue by a [test resource](Test_Skel_Mesh_BindPose_Scale_1.fbx): 

![image-20220412111334664](image-20220412111334664.png)

After being imported to the UE4, the `joint4` ref pose became`(0.333, 0.333, 0.333)`: 

 ![image2022-4-11_21-1-30](image2022-4-11_21-1-30.png)

Oops. 

# Skeletal Mesh Importing 

So the key is in function `UnFbx::FFbxImporter::ImportBone`. Each bone's transform is evaluated in global space first, and then calculate the local transform when building the ref pose transform array. 

```cpp
		//Add the join orientation
		GlobalsPerLink[LinkIndex] = GlobalsPerLink[LinkIndex] * FFbxDataConverter::GetJointPostConversionMatrix();
		if (LinkIndex)
		{
			FbxAMatrix	Matrix;
			Matrix = GlobalsPerLink[ParentIndex].Inverse() * GlobalsPerLink[LinkIndex];
			LocalLinkT = Matrix.GetT();
			LocalLinkQ = Matrix.GetQ();
			LocalLinkS = Matrix.GetS();
		}
		else	// skeleton root
		{
			// for root, this is global coordinate
			LocalLinkT = GlobalsPerLink[LinkIndex].GetT();
			LocalLinkQ = GlobalsPerLink[LinkIndex].GetQ();
			LocalLinkS = GlobalsPerLink[LinkIndex].GetS();
		}
```

It turned out that the `Global Scale Value`, which is calculated by `fbxsdk`, of `Joint4` is `(1, 1, 1)` instead of `(3, 3, 3)`. 

![image2022-4-11_20-41-36](image2022-4-11_20-41-36.png)

And that's the reason why ref pose transform of `Joint4` became `(0.333, 0.333, 0.333)` after the mesh is imported into UE4. 

# Deeper Thought

So why does `Fbxsdk` give `(1, 1, 1)` of global scale value? 

Let's take a look at what a `bind pose` is: 

> The bind pose is the pose that the skeleton is in when you bind skin and so it is the base (or rest) pose of your character.

And that is to say, a bind pose is supposed not to be skinned while being modified. 

You might also notice that the skeletal mesh is not scaled even if it is skinned to a scaled bone: 

![image-20220412174319153](image-20220412174319153.png)

In another word, the ref pose scale value of one bone would be normalized as soon as this bone got skinned with a mesh. 

Actually, there is really no need for us to set a non-identity scale value. 

# Conclusion

In short, don't use a non-identity scale value for bind pose, even if it might look fine in the first place. It may cause some trouble when handle additive animation or skeletal control nodes. 

