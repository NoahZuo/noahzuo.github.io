---
title: How to get rotation from two vectors? 
mathjax: true
tags: 
 - Math
category: Math
date: 2021-02-10 16:40:12
updated: 2021-02-11 12:12:28
description: This article tells what is happening when you try to get a rotation that rotate a vector so that it eventually aligns to another vector. 
---

# Introduction
It is fairly common to calcualte a rotator that rotate a vector $v$ to another vector $u$, especially when we handle something like animation IK. 

Game engines should have their own method for this feature. `Quaternion.FromToRotation()` in Unity, and `FQuat::FindBetweenNormals()` in UE4. But it is always good to know what is actually happending deep inside. 

There are already some really goot posts about this issue you might want to read as well: 
http://lolengine.net/blog/2014/02/24/quaternion-from-two-vectors-final
http://www.euclideanspace.com/maths/algebra/vectors/angleBetween/index.htm

# Math Logic
## A Naive Method
As we all know, a quaternion can be presented by rotating $\theta$ angle around an axis $w$: 

$$q = (\theta, w)$$

Angle can be easily calculated by dot production: 

$$u \cdot v = |u| \cdot |v| \cdot \cos{\theta}$$

$$ || u \times v || = |u| \cdot |v| \cdot \sin{\theta}  $$

```cpp
quat quat::fromtwovectors(vec3 u, vec3 v)
{
    float cos_theta = dot(normalize(u), normalize(v));
    float angle = acos(cos_theta);
    vec3 w = normalize(cross(u, v));
    return quat::fromaxisangle(angle, w);
}
```
It works, in a naive way, though. 

## How To Build a Quaternion? 
It is worth mentioning how a quaterion is built by an axis and an angle. 

Quaternion is constructed by: 

$$ q = \cos{\frac{\theta}{2}} + i\sin{\frac{\theta}{2}}v_{x} + j\sin{\frac{\theta}{2}}v_y + k\sin{\frac{\theta}{2}}v_z$$

So the code for `quat::fromaxisangle` is like this: 
```cpp
quat quat::fromaxisangle(float angle, vec3 axis)
{
    float half_sin = sin(0.5f * angle);
    float half_cos = cos(0.5f * angle);
    return quat(half_cos,
                half_sin * axis.x,
                half_sin * axis.y,
                half_sin * axis.z);
}
```
# Optimization
## Avoiding Trigonometry
Trigonometry is expensive. Thus we need to avoid trigonometry if we can. Inigo Quilez has wrote an article about this: https://www.iquilezles.org/www/articles/noacos/noacos.htm

Sadly, in our naive method, we firstly compute $\theta$ by $\cos{\theta}$, then we compute $\cos{\frac{\theta}{2}}$ and $\sin{\frac{\theta}{2}}$. This is, as a matter of fact,  really expensive. 

Thus we can save performance by [half-angle](https://en.wikipedia.org/wiki/Double-angle_formula#Double-angle.2C_triple-angle.2C_and_half-angle_formulae) by: 
$$\sin{\frac{\theta}{2}} = \sqrt{\frac{1 - \cos{\theta}}{2}}$$ 
$$\cos{\frac{\theta}{2}} = \sqrt{\frac{1 + \cos{\theta}}{2}}$$

So we can save these performance by: 

```cpp
quat quat::fromtwovectors(vec3 u, vec3 v)
{
    float cos_theta = dot(normalize(u), normalize(v));
    float half_cos = sqrt(0.5f * (1.f + cos_theta));
    float half_sin = sqrt(0.5f * (1.f - cos_theta));
    vec3 w = normalize(cross(u, v));
    return quat(half_cos,
                half_sin * w.x,
                half_sin * w.y,
                half_sin * w.z);
}
```
## Avoid One Square Root

Still, we can do better. We need to normalize 3 vectors: `u`, `v` and`cross(u, v)`. But we can get the norm of `w` through: 
$$||u \times v|| = |u| \cdot |v| \cdot |\sin{\theta}| $$
And we know $\sin{\theta}$ by: 
$$\sin{\theta} = 2\sin{\frac{\theta}{2}}\cos{\theta}{2}$$
And we also have : 
$$\sqrt{a} \cdot \sqrt{b} = \sqrt{a \cdot b}$$
So we can save 2 square roots: 
```cpp
quat quat::fromtwovectors(vec3 u, vec3 v)
{
    float norm_u_norm_v = sqrt(sqlength(u) * sqlength(v));
    float cos_theta = dot(u, v) / norm_u_norm_v;
    float half_cos = sqrt(0.5f * (1.f + cos_theta));
    float half_sin = sqrt(0.5f * (1.f - cos_theta));
    vec3 w = cross(u, v) / (norm_u_norm_v * 2.f * half_sin * half_cos);
    return quat(half_cos,
                half_sin * w.x,
                half_sin * w.y,
                half_sin * w.z);
}
```
## Avoid One More Square Root
And we can simplify this by get rid of $\sin{\theta / 2}$: 
```cpp
quat quat::fromtwovectors(vec3 u, vec3 v)
{
    float norm_u_norm_v = sqrt(sqlength(u) * sqlength(v));
    float cos_theta = dot(u, v) / norm_u_norm_v;
    float half_cos = sqrt(0.5f * (1.f + cos_theta));
    vec3 w = cross(u, v) / (norm_u_norm_v * 2.f * half_cos);
    return quat(half_cos, w.x, w.y, w.z);
}
```
This is basically code used by `ORGE3D`

## Consider SIMD
On many SIMD architectures, normalizing a quaterion can be very fast. 

So we should take this advantage: 
```cpp
quat quat::fromtwovectors(vec3 u, vec3 v)
{
    float norm_u_norm_v = sqrt(sqlength(u) * sqlength(v));
    float cos_theta = dot(u, v) / norm_u_norm_v;
    float half_cos = sqrt(0.5f * (1.f + cos_theta));
    vec3 w = cross(u, v) / norm_u_norm_v;
    return normalize(quat(2.f * half_cos * half_cos, w.x, w.y, w.z));
}
```
More over, `half_cos` comes from a `sqrt`, thus we can get rid of this square root : 
```cpp
quat quat::fromtwovectors(vec3 u, vec3 v)
{
    float norm_u_norm_v = sqrt(sqlength(u) * sqlength(v));
    float cos_theta = dot(u, v) / norm_u_norm_v;
    vec3 w = cross(u, v) / norm_u_norm_v;
    return normalize(quat(1.f + cos_theta, w.x, w.y, w.z));
}
```

And for the same reason, we can have: 
```cpp
quat quat::fromtwovectors(vec3 u, vec3 v)
{
    float norm_u_norm_v = sqrt(sqlength(u) * sqlength(v));
    vec3 w = cross(u, v);
    quat q = quat(norm_u_norm_v + dot(u, v), w.x, w.y, w.z);
    return normalize(q);
}
```

## Unit Vector
If vector `u` and `v` are unit vectors, we can have: 
```cpp
quat quat::fromtwovectors(vec3 u, vec3 v)
{
    vec3 w = cross(u, v);
    quat q = quat(1.f + dot(u, v), w.x, w.y, w.z);
    return normalize(q);
}
```

## For Non-Unit Vector
Because we need to calculate $dot(u, v)$ and $cross(u, v)$ no matter what, thus `norm_u_norm_v` can be calculated in a different way: 

```cpp
quat quat::fromtwovectors(vec3 u, vec3 v)
{
    vec3 w = cross(u, v);
    quat q = quat(dot(u, v), w.x, w.y, w.z);
    q.w += length(q);
    return normalize(q);
}
```

We can get rid of at least 3 multiplications in this way. 


# Final Version
I just copy code from : http://www.euclideanspace.com/maths/algebra/vectors/angleBetween/index.htm

```cpp
/** note v1 and v2 dont have to be nomalised, thanks to minorlogic for telling me about this:
* https://www.euclideanspace.com/maths/algebra/vectors/angleBetween/minorlogic.htm
*/
sfquat angleBetween(sfvec3f v1,sfvec3f v2) { 
	float d = sfvec3f.dot(v1,v2); 
	sfvec3f axis = v1;
	axis.cross(v2);
    float qw = (float)Math.sqrt(v1.len_squared()*v2.len_squared()) + d;
	if (qw < 0.0001) { // vectors are 180 degrees apart
		return (new sfquat(0,-v1.z,v1.y,v1.x)).norm;
	} 
	sfquat q= new sfquat(qw,axis.x,axis.y,axis.z); 
    return q.norm();
} 

```



