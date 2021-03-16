---
title: Some Math About Capsule Collision
mathjax: true
tags: 
 - Math
 - Physics
category: Math
date: 2021-03-15 15:29:03
updated: 2021-03-16 17:19:56
description: This article records something about capsule collision calculation.  

---

# Introduction
## What Is A Capsule? 
The capsule collision is a simple extension to the sphere intersection test. A capsule is defined, in most cases, by a base point `a` and a tip point `b`. 

A capsule collision can be defined like this: 
$$
R = \{x|(x - [a + (b - a) * t]) ^ 2 \leq r \}, 0 \leq t \leq 1
$$

![image-20210315163352510](image-20210315163352510.png)

## Why Is Capsule Important? 
We can of course use cylinders as bounding volumes. Unfortunately, it turns out that the overlap test for cylinders is quite expensive. 


As an alternative result, capsule shapes are wildly used in physics engines because of calculation simplicity. Thus it is always good to know what is happening deep inside when collisions are detected. 

# Collision Detection Methods

## Sphere-Capsule Intersection

A useful property of capsules, or rather, sphere-swept volumes, is that the distance computation between the inner structures does not rely on the inner structures being of the same type. 

So we only need to compare the shortest distance and the radius sum, and that's it. 

![image-20210315174151863](image-20210315174151863.png)



The code is like this: 

```cpp
int TestSphereCapsule(Sphere s, Capsule capsule)
{
    // Compute (squared) distance between sphere center and capsule line segment
    float dist2 = SqDistPointSegment(capsule.a, capsule.b, s.c);
    // If (squared) distance smaller than (squared) sum of radii, they collide
    float radius = s.r + capsule.r;
    return dist2 <= radius * radius;
}
```

Distance from point to a segment can be easily calculated by: 

```cpp
// Returns the squared distance between point c and segment ab
float SqDistPointSegment(Point a, Point b, Point c)
{
    Vector ab = b – a, ac = c – a, bc = c – b;
    float e = Dot(ac, ab);
    // Handle cases where c projects outside ab
    if (e <= 0.0f) return Dot(ac, ac);
    float f = Dot(ab, ab);
    if (e >= f) return Dot(bc, bc);
    // Handle cases where c projects onto ab
    return Dot(ac, ac) – e * e / f;
}
```

Or you may want the reference point value: 

```cpp
void ClosestPtPointSegment(Point c, Point a, Point b, float &t, Point &d)
{
    Vector ab = b – a;
    // Project c onto ab, but deferring divide by Dot(ab, ab)
    t = Dot(c – a, ab);
    if (t <= 0.0f) {
        // c projects outside the [a,b] interval, on the a side; clamp to a
        t = 0.0f;
        d = a;
    } else {
        float denom = Dot(ab, ab); // Always nonnegative since denom = ||ab||∧2
        if (t >= denom) {
        // c projects outside the [a,b] interval, on the b side; clamp to b
        t = 1.0f;
        d = b;
        } else {
            // c projects inside the [a,b] interval; must do deferred divide now
            t = t / denom;
            d = a + t * ab;
        }
    }
}
```





## Capsule-Capsule Intersection

It is natural to consider a capsule-capsule intersection as a problem to solve the distance between two segments: 

```cpp
int TestCapsuleCapsule(Capsule capsule1, Capsule capsule2)
{
    // Compute (squared) distance between the inner structures of the capsules
    float s, t;
    Point c1, c2;
    float dist2 = ClosestPtSegmentSegment(capsule1.a, capsule1.b,
    capsule2.a, capsule2.b, s, t, c1, c2);
    // If (squared) distance smaller than (squared) sum of radii, they collide
    float radius = capsule1.r + capsule2.r;
    return dist2 <= radius * radius;
}
```

This is much more complicated, however, to compute closest points of two segments. 

I simply copy the code from *Real Time Collision Detection* here since this is not the point in this article: 

```cpp
// Computes closest points C1 and C2 of S1(s)=P1+s*(Q1-P1) and
// S2(t)=P2+t*(Q2-P2), returning s and t. Function result is squared
// distance between between S1(s) and S2(t)
float ClosestPtSegmentSegment(Point p1, Point q1, Point p2, Point q2,
float &s, float &t, Point &c1, Point &c2)
{
    Vector d1 = q1 - p1; // Direction vector of segment S1
    Vector d2 = q2 - p2; // Direction vector of segment S2
    Vector r = p1 - p2;
    float a = Dot(d1, d1); // Squared length of segment S1, always nonnegative
    float e = Dot(d2, d2); // Squared length of segment S2, always nonnegative
    float f = Dot(d2, r);
    // Check if either or both segments degenerate into points
    if (a <= EPSILON && e <= EPSILON) {
        // Both segments degenerate into points
        s = t = 0.0f;
        c1 = p1;
        c2 = p2;
        return Dot(c1 - c2, c1 - c2);
    }
    if (a <= EPSILON) {
        // First segment degenerates into a point
        s = 0.0f;
        t = f / e; // s = 0 => t = (b*s + f) / e = f / e
        t = Clamp(t, 0.0f, 1.0f);
    } else {
        float c = Dot(d1, r);
        if (e <= EPSILON) {
            // Second segment degenerates into a point
            t = 0.0f;
            s = Clamp(-c / a, 0.0f, 1.0f); // t = 0 => s = (b*t - c) / a = -c / a
        } else {
            // The general nondegenerate case starts here
            float b = Dot(d1, d2);
            float denom = a*e-b*b; // Always nonnegative
            // If segments not parallel, compute closest point on L1 to L2 and
            // clamp to segment S1. Else pick arbitrary s (here 0)
            if (denom != 0.0f) {
                s = Clamp((b*f - c*e) / denom, 0.0f, 1.0f);
            } else s = 0.0f;
            // Compute point on L2 closest to S1(s) using
            // t = Dot((P1 + D1*s) - P2,D2) / Dot(D2,D2) = (b*s + f) / e
            t = (b*s + f) / e;
            // If t in [0,1] done. Else clamp t, recompute s for the new value
            // of t using s = Dot((P2 + D2*t) - P1,D1) / Dot(D1,D1)= (t*b - c) / a
            // and clamp s to [0, 1]
            if (t < 0.0f) {
                t = 0.0f;
                s = Clamp(-c / a, 0.0f, 1.0f);
            } else if (t > 1.0f) {
                t = 1.0f;
                s = Clamp((b - c) / a, 0.0f, 1.0f);
            }
        }
    }
    c1 = p1 + d1 * s;
    c2 = p2 + d2 * t;
    return Dot(c1 - c2, c1 - c2);
}
```



## Triangle-Capsule Intersection

Still we need to talk about triangle-capsule intersection. 

First of all, we can find the closest point on a line to this triangle. 

![image-20210315210702293](image-20210315210702293.png)



```cpp
float3 ClosestPointOnLineSegment(float3 A, float3 B, float3 Point)
{
  float3 AB = B – A;
  float t = dot(Point – A, AB) / dot(AB, AB);
  return A + saturate(t) * AB; // saturate(t) can be written as: min((max(t, 0), 1)
}


// Compute capsule line endpoints A, B like before in capsule-capsule case:
float3 CapsuleNormal = normalize(tip – base); 
float3 LineEndOffset = CapsuleNormal * radius; 
float3 A = base + LineEndOffset; 
float3 B = tip - LineEndOffset;
 
// Then for each triangle, ray-plane intersection:
//  N is the triangle plane normal (it was computed in sphere – triangle intersection case)
float t = dot(N, (p0 - base) / abs(dot(N, CapsuleNormal)));
float3 line_plane_intersection = base + CapsuleNormal * t;
 
float3 reference_point = {find closest point on triangle to line_plane_intersection};
 
// The center of the best sphere candidate:
float3 center = ClosestPointOnLineSegment(A, B, reference_point);
```



And this is how we find the reference point: 

```cpp
// Determine whether line_plane_intersection is inside all triangle edges: 
float3 c0 = cross(line_plane_intersection – p0, p1 – p0) 
float3 c1 = cross(line_plane_intersection – p1, p2 – p1) 
float3 c2 = cross(line_plane_intersection – p2, p0 – p2)
bool inside = dot(c0, N) <= 0 && dot(c1, N) <= 0 && dot(c2, N) <= 0;
 
if(inside)
{
  reference_point = line_plane_intersection;
}
else
{
  // Edge 1:
  float3 point1 = ClosestPointOnLineSegment(p0, p1, line_plane_intersection);
  float3 v1 = line_plane_intersection – point1;
  float distsq = dot(v1, v1);
  float best_dist = distsq;
  reference_point = point1;
 
  // Edge 2:
  float3 point2 = ClosestPointOnLineSegment(p1, p2, line_plane_intersection);
  float3 v2 = line_plane_intersection – point2;
  distsq = dot(vec2, vec2);
  if(distsq < best_dist)
  {
    reference_point = point2;
    best_dist = distsq;
  }
 
  // Edge 3:
  float3 point3 = ClosestPointOnLineSegment(p2, p0, line_plane_intersection);
  float3 v3 = line_plane_intersection – point3;
  distsq = dot(v3, v3);
  if(distsq < best_dist)
  {
    reference_point = point3;
    best_dist = distsq;
  }
}
```

