---
title: What is motion fields for character animation? 
tags: 
 - Animation
 - Programming
 - Motion matching
category: Animation
description: This article tells you something about motion fields, a method for motion matching. 

---

I have been learning motion matching for quite a few days. Motion matching is the next-gen character animation controller and it can produce great character locomotion without handling huge amount of state machines. 

## Traditional Animation Controller in Games

How do we achieve character locomotion using traditional approach? 

Animators would like to produce animation raw resource(.fbx files). We import those animation resources into game engine and blend spaces(for UE4) or blendtree(for Unity) would be created. 

I'll use Unity for illustration. Blendtree is driven by one or two animator parameters(I have implemented RBF blending, which allow us to drive blendtree using more than 2 parameters, in Unity though), we can modify those parameters in our gameplay logic to control final animation effect. 

For character locomotion, the most common approach is to setup an 8-direction blendtree. As a result, we will drive the blend tree using character moving direction directly. 

In 10 years ago, this might be acceptable. But since it is 2019, something still needs to be done. 

If we need to add some decorations to the 8-direction blendtree, we'll have to add so many states like `Pre pivot` and `Post pivot`. 

Moreover, we need to handle those complex transitions. And the complexity grows rapidly as states grow. 

So... 

