---
title: 优化Unity中ParticleControlPlayable在微信小游戏平台的CPU开销
tags: 
 - Unity
 - 微信小游戏
category: Unity
date: 2025-03-31 16:58:14

description: 这篇文章记录了Unity的微信小游戏平台工程中，针对在`Timeline`里控制的粒子系统的CPU热点进行优化的流程介绍。

---

# 背景

Unity官方提供了[Timeline工具](https://docs.unity3d.com/Packages/com.unity.timeline@1.2/manual/index.html)，用于进行场景内物件的控制，对标于Unreal Engine的[Level Sequence](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-sequencer-movie-tool-overview)。

但是当Timeline中出现了太多Particle时，则很容易出现Simulate的性能瓶颈，该问题在微信小游戏平台中更为严重，本文记录了针对该性能热点进行优化的思考和流程。

# 热点介绍

构建一个Unity工程，在Timeline中添加20个`Control Track`，将其控制场景中的各类粒子：

![Timeline设置](image01.png)

使用微信小游戏快适配插件构建小游戏包，记得勾选`Profiling Funcs`，方便我们查看CPU性能的开销：

![Profiling Funcs](image02.png)

在安卓机上进行真机预览，点击`Start/Stop CPU Profile`进行性能录制，之后便可以在`Android/Data/com.tencent.mm/MicroMsg/appbrand/trace`目录下找到cpuprofile文件：

![Start Or Stop CPU Profile](image03.png)

该文件可使用Google Chrome打开，我们看到，Particle开销如下：
![Particle Performance](image04.png)

# 源码分析

参考官方Timeline插件相关的代码：

```cs
public override void PrepareFrame(Playable playable, FrameData data)
{
    if (particleSystem == null || !particleSystem.gameObject.activeInHierarchy)
    {
        // case 1212943
        m_LastPlayableTime = kUnsetTime;
        return; 
    }

    var time = (float)playable.GetTime();
    var particleTime = particleSystem.time;

    // if particle system time has changed externally, a re-sync is needed
    if (m_LastPlayableTime > time || !Mathf.Approximately(particleTime, m_LastParticleTime))
        Simulate(time, true);
    else if (m_LastPlayableTime < time)
        Simulate(time - m_LastPlayableTime, false);

    m_LastPlayableTime = time;
    m_LastParticleTime = particleSystem.time;
}

private void Simulate(float time, bool restart)
{
    const bool withChildren = false;
    const bool fixedTimeStep = false;
    float maxTime = Time.maximumDeltaTime;

    if (restart)
        particleSystem.Simulate(0, withChildren, true, fixedTimeStep);

    // simulating by too large a time-step causes sub-emitters not to work, and loops not to
    // simulate correctly
    while (time > maxTime)
    {
        particleSystem.Simulate(maxTime, withChildren, false, fixedTimeStep);
        time -= maxTime;
    }

    if (time > 0)
        particleSystem.Simulate(time, withChildren, false, fixedTimeStep);
}

```

可以看到，主要的开销最终还是落在了`particleSystem.Simulate`函数中。

# 优化方案

既然已经定位到了性能热点来自`particleSystem.Simulate`，因此优化的方式便集中于怎样减少该函数的调用。以下提出若干种优化的方案，希望对于读者有帮助。

## 根据屏占比降低更新频率

第一个想到的方案自然是根据Particle的屏占比来决定该粒子的更新频率。首先我们需要进行Particle System的屏占比计算。笔者提供了一个基于Bounding Sphere的屏占比计算方法：

```cs
private float CalculateScreenCoverageRatio()
{
    if (m_MainCamera == null)
        return 1.0f;

    // Get particle system bounds
    Bounds bounds = particleSystem.GetComponent<Renderer>().bounds;
    Vector3 center = bounds.center;
    float radius = bounds.extents.magnitude;

    // Project bounding sphere to screen space
    Vector3 screenCenter = m_MainCamera.WorldToViewportPoint(center);
    if (screenCenter.z < 0) // Behind camera
        return 0f;

    // Calculate screen space radius
    Vector3 screenEdge = m_MainCamera.WorldToViewportPoint(center + m_MainCamera.transform.right * radius);
    float screenRadius = (screenEdge - screenCenter).magnitude;

    // Calculate coverage ratio (clamped to 0-1)
    float coverage = Mathf.Clamp01(screenRadius * 2); // Diameter coverage
    return coverage * coverage; // Square for non-linear response
}
```
之后，我们只需要`PrepareFrame`函数中添加针对屏占比降频的操作即可。在此，我们根据屏占比分了三档——每帧更新、隔一帧更新和隔两帧更新：

```cs
private int m_FrameCount = 0;
private int m_SkipFrames = 0;
private Camera m_MainCamera;
private const float HighThreshold = 0.5f;  // Update every frame
private const float MidThreshold = 0.2f;   // Update every other frame

public override void PrepareFrame(Playable playable, FrameData data)
{
    if (particleSystem == null || !particleSystem.gameObject.activeInHierarchy)
    {
        m_LastPlayableTime = kUnsetTime;
        return;
    }

    // Get main camera if not cached
    if (m_MainCamera == null)
        m_MainCamera = Camera.main;

    var time = (float)playable.GetTime();
    var particleTime = particleSystem.time;
    
    // Calculate screen coverage ratio
    float screenRatio = CalculateScreenCoverageRatio();
    
    // Determine update frequency based on screen coverage
    int requiredSkipFrames = screenRatio > HighThreshold ? 0 : 
                            screenRatio > MidThreshold ? 1 : 2;
    
    // Reset frame count when frequency changes
    if (m_SkipFrames != requiredSkipFrames)
    {
        m_SkipFrames = requiredSkipFrames;
        m_FrameCount = 0;
    }

    // Check if we need to update
    bool forceUpdate = m_LastPlayableTime > time || !Mathf.Approximately(particleTime, m_LastParticleTime);
    bool shouldUpdate = forceUpdate || (m_FrameCount % (m_SkipFrames + 1)) == 0;

    if (shouldUpdate)
    {
        float deltaTime = forceUpdate ? time : time - m_LastPlayableTime;
        
        if (m_LastPlayableTime > time || !Mathf.Approximately(particleTime, m_LastParticleTime))
            Simulate(time, true);
        else if (m_LastPlayableTime < time)
            Simulate(deltaTime, false);

        m_LastPlayableTime = time;
        m_LastParticleTime = particleSystem.time;
    }

    // Increment frame counter (wrap to prevent overflow)
    m_FrameCount = (m_FrameCount + 1) % int.MaxValue;
}
```

使用该方法进行性能分析，得到结果如下：

![根据屏占比降频](image05.png)

这种做法实现简单，并且在低端机的优化效果也很明显，但是问题也依然明显：**对视觉效果影响过大**。

举例：如果当前场景整体运算负载较低，足够游戏满帧运行，但是屏占比较低的粒子依然进行了降频，从而影响了视觉效果。

## 根据帧耗时进行动态频率控制

一种更好的做法则为根据帧耗时进行动态的粒子更新频率控制，也就是说只有在负载比较高的时候才进行频率的控制，参考代码如下：

```cs
private const float FrameTimeHigh = 16f; // 60 FPS threshold
private const float FrameTimeLow = 25f;  // 40 FPS threshold
private float m_FrameTimeEMA; // Exponential moving average of frame time

public override void PrepareFrame(Playable playable, FrameData data)
{
    if (particleSystem == null || !particleSystem.gameObject.activeInHierarchy)
    {
        m_LastPlayableTime = kUnsetTime;
        return;
    }

    // Get main camera if not cached
    m_MainCamera ??= Camera.main;

    var time = (float)playable.GetTime();
    var particleTime = particleSystem.time;
    
    // 1. Calculate screen coverage base tier
    float coverage = CalculateScreenCoverageRatio();
    int baseSkipFrames = coverage switch
    {
        > 0.5f => 0,  // High visibility
        > 0.2f => 1,  // Medium visibility
        _      => 2   // Low visibility
    };

    // 2. Adjust by frame time
    float currentFrameTime = Time.unscaledDeltaTime * 1000; // Current frame time in ms
    m_FrameTimeEMA = 0.8f * m_FrameTimeEMA + 0.2f * currentFrameTime; // EMA smoothing
    
    int adjustedSkipFrames = m_FrameTimeEMA switch
    {
        < FrameTimeHigh => Mathf.Max(baseSkipFrames - 1, 0), // Good performance: higher quality
        > FrameTimeLow  => Mathf.Min(baseSkipFrames + 1, 3), // Poor performance: lower quality
        _               => baseSkipFrames                    // Normal: use base setting
    };

    // 3. Update frequency control
    if (m_SkipFrames != adjustedSkipFrames)
    {
        m_SkipFrames = adjustedSkipFrames;
        m_FrameCount = 0;
    }

    // 4. Update logic
    bool needsForceUpdate = m_LastPlayableTime > time || 
                          !Mathf.Approximately(particleTime, m_LastParticleTime);
    bool shouldUpdate = needsForceUpdate || 
                      (m_FrameCount % (m_SkipFrames + 1) == 0);

    if (shouldUpdate)
    {
        if (needsForceUpdate)
            Simulate(time, true);
        else if (m_LastPlayableTime < time)
            Simulate(time - m_LastPlayableTime, false);

        m_LastPlayableTime = time;
        m_LastParticleTime = particleSystem.time;
    }

    // 5. Maintain counters
    m_FrameCount = (m_FrameCount + 1) % int.MaxValue;
}
```

这种方案需要结合帧耗时与屏占比两个指标，从而实现起来较为复杂，并且效果也没有那么直观，性能优化结果如下：

![帧耗时+屏占比](image06.png)

## Simulation Budget系统

有一种最为理想的做法则是Simulation Budget系统，该系统参考自Unreal Engine的[Animation Budget系统](https://dev.epicgames.com/documentation/en-us/unreal-engine/animation-budget-allocator-in-unreal-engine)。

简单来说，该系统针对某个模块在某一帧分配了一个相对固定的开销预算（该预算值可以针对不同档位的机器进行配置），并且维护了每个粒子的动态优先级，该优先级计算方案如下：
 - 如果屏占比越高，则优先级越高；
 - 如果该粒子上一帧没有进行Simulate操作，则本帧该粒子的优先级变高；

使用这种方案有几种好处：
 1. 管理成本较低，针对不同档位的机型只需要配置开销的预算即可。
 2. 兼顾表现与性能，在负载较低时能确保表现，在负载较高时能减少卡顿。
 3. 开销曲线较为平稳。

该方案的实现需要通过`Stopwatch`获取到每个粒子的`Simulate`开销，从而判断剩余的性能开销预算是否足够支撑接下来的粒子模拟。

但是由于微信小游戏环境中，高精度时间只能在安卓和iOS普通模式下起效，因此通常的`particleSystem.Simulate`方法的执行时间都会被判定为`0ms`，因此该方案理论上只能在安卓平台上起效。