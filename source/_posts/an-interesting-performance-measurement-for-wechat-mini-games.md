---
title: 微信小游戏中函数开销评估的一种高效方法
mathjax: true
tags: 
 - 微信小游戏
 - Unity
category: 微信小游戏
date: 2025-05-12 19:22:29
updated: 2025-05-13 20:13:57
description: 本文介绍了一种在微信小游戏中以较高精度评估函数执行效率的方法，通过统计校正避免了采样误差。
---

# 背景说明

微信小游戏提供了基于Android CPU Profiler的功能，该功能允许在固定频率下对函数栈进行采样，并生成相应的Timeline火焰图。

![火焰图示例](image.png)

在火焰图中，每次函数栈采样会被统计为0.17ms的开销，从而定位性能热点。然而，由于该功能基于采样，难以避免偶然性问题。例如，某个函数的实际执行时间可能远低于0.17ms，但某次采样恰好捕获到该函数，导致其开销被误判为0.17ms。

![采样误差示例](image2.png)

此外，这种视图下，帧与帧之间的Profile结果可能存在较大偏差，进一步扩大了误差范围。这对微信小游戏的性能优化工作带来了一定干扰。

# 方法思路

该问题的本质是低频稀疏事件在固定间隔采样下的统计高估现象。解决方案可以从以下三个方向考虑：

1. **提高采样频率**：
   - 代价：采样开销增加，数据量爆炸，不现实。

2. **结合插桩（Instrumentation）**：
   - 在关键函数中插入计时代码，直接测量函数开销，消除采样误差。
   - 代价：代码侵入性强，额外性能开销，且仅适用于已知函数，不现实。

3. **统计校正**：
   - 若已知函数调用次数`N`，可通过统计数学方法进行校正。
   - 例如，基于[SpeedScope](https://www.speedscope.app/)的Left Heavy功能，可以获取函数采样的总耗时。
   - 若能通过某种方式获取调用次数`N`，即可最大程度消除统计偏差。

# 实现细节

## 固定帧开销

我们添加一个`MonoBehaviour`脚本，在`Update`函数中执行固定时间开销的运算，代码如下：

```cs
// Update函数每帧调用一次
void Update()
{
    // 使用Stopwatch计时，避免Thread.Sleep（微信小游戏中无效）
    Stopwatch sw = new Stopwatch();
    sw.Start();

    // 确保每帧运行时间为10ms，便于在CPU Profiler中对比
    while (sw.ElapsedMilliseconds <= 10)
    {
        float result = 0;
        for (int i = 0; i < 100; i++)
        {
            float tmpi = i;
            result += Mathf.Sin(tmpi * tmpi * 0.95f);
        }
    }
    sw.Stop();
}
```

需要注意的是，Web环境下默认时间精度为1ms，容易引入误差。因此，我们需要启用高精度时间功能。在`weapp-adapter.js`中修改`clientPerfAdapter`如下：

```javascript
const clientPerfAdapter = Object.assign({}, {
    now: function now() {
        if (!GameGlobal.unityNamespace.isDevtools
            && !GameGlobal.isIOSHighPerformanceMode) {
            // wx.getPerformance()返回微秒级时间，需除以1000统一单位
            return (ori_performance.now() - ori_initTime) * 0.001;
        }
        return (Date.now() - initTime);
    },
});
```

## 数据分析

在安卓真机测试中采集`.cpuprofile`文件后，使用`SpeedScope`的`Left Heavy`页面可以获取`Update`函数的总调用时间。

![数据分析示例](image3.png)

假设我们需要分析`Animator::BatchedIKPass`的开销，由于`Update`函数的每帧执行时间是稳定的，可以列出以下方程：

$$ \frac {UIEventManager.Update}{Animator.BatchedIKPass} = \frac{10ms}{x_{batchedikpass}} $$

解得 $x_{batchedikpass} = 4.1$ms。

通过这种方式，我们可以在采样Profiling体系下获取目标函数的平均开销，同时避免单帧数据的偶然性。