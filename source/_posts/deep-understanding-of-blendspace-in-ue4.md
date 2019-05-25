---
title: Deep Understanding of Blend Space in UE4
tags: 
 - UE4
 - Animation
 - Programming
category: UE4
description: This article gives a underlying implementation of animation blend space of UE4. 
---

Epic has already give [documentation about blend spaces in UE4](https://docs.unrealengine.com/en-us/Engine/Animation/Blendspaces), though. It is necessary for me to get to know what is happening in low-level when we run a blend space in an animation blueprint. 



## What is Happening in Blend Space Editor? 

When we are going to play with a blend space, we have to set up the horizontal&vertical axis settings in Editor. And I assume you have basic knowledge about blend space editor, if not, please refer to the [official documentation](https://docs.unrealengine.com/en-US/engine/animation/blendspaces/editor). 

In general, you can setup how final pose is interpolated between animations in the Grid Editor: 

![Grid Option](GridOptionsImage.png)

After you have done with `Axis Settings` and `Blend Samples`, or modify those settings, data would be re-sampled and grid elements would be re-generated. 

### Triangulation

If you hit the `Show Triangulation` button, you can see that all sample points forming a triangle list: 

![Triangulation List](TriangulationList.png)

UE4 use [Delaunay triangulation](https://en.wikipedia.org/wiki/Delaunay_triangulation) for triangulation because we need to interpolate inside a triangle. And Delaunay triangulation tend to avoid sliver triangles. I will post an article about Delaunay triangulation some day. 

![Delaunay Triangulation](Delaunay Triangulation.png)

### Generate Grid Element Information

We need to calculate additional information for each grid point after triangulation. In particular, index and weight of each vertice of the triangle that this grid point locates. 

![Grid Info VS](Grid Info.png)

![Grid Info Engine](Grid Info2.png)

As you can see in the image above, each grid point contains index and weight information. Such information would be stored along with asset. 

## How are Animations Sampled? 

We need a 2D coordinate to tell the blend space what we want. But how on earth are those animation sampled? How to get the weight of each animation? 

![Sample Weights](Samples.png)

The calculation is truly easy. Firstly we'll determine which grid our blend position locate in: 

![Grid we need](Grid We Look For.png)

Then we can get each point's information like what we explained above. Then we can easily bi-interpolate between these values, and get the final sample weights we need. 



## Blend Space Interpolation

A lot of work has been done in UE4 to give a good blending effect between animations. 

### Axis Interpolation

UE4 allows us to interpolate our axis parameter value. If Interpolation Time has inputs, the real blend position would be a interpolation between our old sample value and the new value. 

![Axis interpolation](axis interpolation.png)

### Target Weight Interpolation

Target weight interpolation is a fantastic feature for locomotion. This allows us to directly interpolate between weights instead of parameters. 

![Target Weight Interpolation](target weight interpolation.png)

For example, if we interpolate input, when we move from left to right rapidly, we'll interpolate through forward animation, and this might not be what we want. 

If we use target weight interpolation, forward animation would be skipped, and it will look much better! 

And actually this is what Unity need most. I have add this feature into our customized Unity Engine, and it works in _Call of Duty Mobile_ and _Cross Fire Mobile_, it works great! 