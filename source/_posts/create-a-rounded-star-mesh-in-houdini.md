---
title: Create a Rounded Star Mesh in Houdini
tags: 
 - Houdini
 - PCG
category: Houdini
date: 2019-05-18 15:53:17
description: This article briefly introduce how to create a rounded star mesh, and export it as a fbx file in houdini. 
---

So... How to create a rounded star mesh in Houdini? 

1. Add a `geometry` node, delete original `file` node. 
2. create a `circle` node, Change `primitive type` to `polygon`: 
   ![Circle Node](circle Node.png)
3. Add a `group range` node and wire the `circle` node into it, set the `group` to `even`, `group type` to `Points`, `Range filter` to `Select 1 of 2`: 
   ![Group by range](groupbyrange.png)
4. Add a `Transform` node and wire the `group range` node into it, set the `Group` to `even` and set the `Uniform scale` to `0.5`:
   ![Transform](transform.png)
5. Add a `Divide` node and wire the `Transform` node into it. Now we have a final rounded star mesh and are ready to export it as a .fbx file: 
   ![Divide](divide.png)