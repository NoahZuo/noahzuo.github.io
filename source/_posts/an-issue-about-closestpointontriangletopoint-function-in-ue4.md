---

title: An Issue About FMath::ClosestPointOnTriangleToPoint Function In UE4
tags: 
 - UE4
 - Math
category: UE4
date: 2022-04-18 15:38:12
description: This article introduces an issue about `FMath::ClosestPointOnTriangleToPoint` in UE4. And implementation methods of `Chaos` and `PhysX` is also introduced in this article. 

---

# Introduction

UE4 has already provide a function `FMath::ClosestPointOnTriangleToPoint` for us to calculate the closest point on a triangle to a specific point: 

```cpp
FVector FMath::ClosestPointOnTriangleToPoint(const FVector& Point, const FVector& A, const FVector& B, const FVector& C)
{
        //Figure out what region the point is in and compare against that "point" or "edge"
        const FVector BA = A - B;
        const FVector AC = C - A;
        const FVector CB = B - C;
        const FVector TriNormal = BA ^ CB;

        // Get the planes that define this triangle
        // edges BA, AC, BC with normals perpendicular to the edges facing outward
        const FPlane Planes[3] = { FPlane(B, TriNormal ^ BA), FPlane(A, TriNormal ^ AC), FPlane(C, TriNormal ^ CB) };
        int32 PlaneHalfspaceBitmask = 0;

        //Determine which side of each plane the test point exists
        for (int32 i=0; i<3; i++)
        {
                if (Planes[i].PlaneDot(Point) > 0.0f)
                {
                        PlaneHalfspaceBitmask |= (1 << i);
                }
        }

        FVector Result(Point.X, Point.Y, Point.Z);
        switch (PlaneHalfspaceBitmask)
        {
        case 0: //000 Inside
                return FVector::PointPlaneProject(Point, A, B, C);
        case 1:        //001 Segment BA
                Result = FMath::ClosestPointOnSegment(Point, B, A);
                break;
        case 2:        //010 Segment AC
                Result = FMath::ClosestPointOnSegment(Point, A, C);
                break;
        case 3:        //011 point A
                return A;
        case 4: //100 Segment BC
                Result = FMath::ClosestPointOnSegment(Point, B, C);
                break;
        case 5: //101 point B
                return B;
        case 6: //110 point C
                return C;
        default:
                UE_LOG(LogUnrealMath, Log, TEXT("Impossible result in FMath::ClosestPointOnTriangleToPoint"));
                break;
        }

        return Result;
}
```

This function works fine for a `right triangle` or an `acute triangle`. But this function can return a wrong result for an [obtuse triangle](https://en.wikipedia.org/wiki/Acute_and_obtuse_triangles): 

![image-20220418200108252](image-20220418200108252.png)

# The Core Algorithm

## How Does This Function Work? 

This function simply split the 3-d space into 8 regions. And the closest point is calculated considering which region it is located in: 

![image-20220418205901968](image-20220418205901968.png)

It's easy to tell that this can perfectly do the trick for an acute triangle through a little calculation.  But this does not work for an obtuse triangle. 

## What About An Obtuse Triangle? 

Imagine a triangle and a point like this: 

![image-20220418203405725](image-20220418203405725.png)

The function can return point `A` as the result of the input point `P`. But actually it is point `D` that is the true closest point on the triangle like this: 

![image-20220418213114046](image-20220418213114046.png)

And that is the issue, or rather, the bug of this implementation. 

# PhysX's Implementation

`PhysX`'s implementation is like this: 

![image-20220419101806005](image-20220419101806005.png)

Two additional dot production comparisons are performed to fix this issue. So the space partition is like this: 

![image-20220419123932918](image-20220419123932918.png)

A precise solution is necessary for `physx` since `GJK algorithm` needs to find the support point. 

# Chaos' Implementation

Chaos has its own implementation as well: 

![image-20220419170139893](image-20220419170139893.png)

This implementation method is quite simple: 

1. Firstly, calculate the target point onto the plane `P'`; 
2. Second, calculate the [barycentric coordinate](https://en.wikipedia.org/wiki/Barycentric_coordinate_system#Conversion_between_barycentric_and_Cartesian_coordinates) of `P'`. If `P'` is located in this triangle, then `P'` is the closest point. 
3. Or else, the result point must locate in the edge of this triangle. Thus, calculate the closest point to these 3 line segment, and find the closest one. 

The performance implementation is not that good. The implementation of `PhysX` is preferred considering performance. 