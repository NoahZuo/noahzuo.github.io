---
title: Better Animation Transition Using Sync Group in Paragon
tags: 
 - UE4
 - Animation
category: Animation
date: 2019-12-29 15:47:12
description: This article briefly tells you how Paragon use Sync Group to achievement a better animation transition. 
---

Paragon is one of those games that has next-gen animation quality that seems to pull a rabbit out of a hat. It is very nice of the develop team to share their techniques on Youtube. One of the great techniques to perform a better animation transition is Sync Group which are integrated into UE4 in 4.11. 

## Traditional Way

Think about a character that is performing a `idle-walk-stop` loop. In the old days we simply do a cross-fade between these animations. This method often results in some unrealistic motions like this: 

![trandition-way](trandition-way.gif)

It is very clear that foot are sliding on the ground. This might seems okay a decade years ago, but nowadays we might want a better solution(~~Bethesda: Excuse me?~~ ). 

Let's try to make jog-start to jog-loop transition: 

In the old days, we need to export two end-to-end jog animations. And we can only jump to the jog-loop animation as soon as the jog-start animation ends. 

But since Paragon is a designer-driven project. More than lots of stuff needs to be modified by designers whenever they want. If the designer want the character to jump into jog-loop state earlier, both animations need to be modified to avoid strange transition like this: 

![pre-jog-strange](prejog2.gif)

It would be wonderful if we can perform the transition at any time. And this is why we need sync group. 

## What Is A Sync Group? 

A sync group is a manager that handles the synchronization of a bunch of animation sequences. 

### Sync Marker

We can place sync markers in notifies panel. Paragon places sync markers in the jog-start and jog-loop animation at the time when left or right foot cross the hip bone. 

![sync marker](sync-marker.gif)

Aside from foot-plant stuff, you can of course add sync markers to some other animations like swinging arms or shaking head. 

### Sync Group

After taking care of sync markers, we need to put those asset player node in a sync group to make it work like this: 

![sync group](sync-group.png)



And this is what looks like: 

![final](sync-marker-final.gif)