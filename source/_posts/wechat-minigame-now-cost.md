---
title: 微信小游戏中的时间戳
mathjax: false
tags: 
 - 微信小游戏
 - Unity
category: 微信小游戏
date: 2025-02-28 12:24:12
description: 这篇文章介绍在Unity微信小游戏中时间戳的相关内容。
---

# 背景

游戏开发中，获取当前的时间戳是极为常用的操作。但是浏览器环境下，wasm本身是无法获取系统时间的，因而必须通过胶水层来实现目的。

本文将介绍微信小游戏环境的时间戳相关的内容，希望对读者有帮助。

# 胶水层主入口

众所周知，在Javascript中获取当前时间的方式有`Date.now()`和`performance.now()`两种方式。在微信小游戏的胶水层中，打开`weapp-adapter.js`文件即可看到胶水层是怎样Hook原有`performance.now()`函数的，摘抄代码如下：

```javascript

    /***/ (function (module, exports) {
        'use strict';
        Object.defineProperty(exports, '__esModule', {
            value: true,
        });
        let performance = void 0;
        let ori_performance = wx.getPerformance();
        const initTime = Date.now();
        const ori_initTime = ori_performance.now();
        const clientPerfAdapter = Object.assign({}, {
            now: function now() {
                if (GameGlobal.unityNamespace.isDevelopmentBuild
                    && GameGlobal.unityNamespace.isProfilingBuild
                    && !GameGlobal.unityNamespace.isDevtools
                    && !GameGlobal.isIOSHighPerformanceMode) {
                    // 由于wx.getPerformance()获取到的是微秒级，因此这里需要/1000.0，进行单位的统一
                    return (ori_performance.now() - ori_initTime) * 0.001;
                }
                return (Date.now() - initTime);
            },
        });
        performance = clientPerfAdapter;
        exports.default = performance;
        /***/ 
    }),
```

可以看到，`performance.now()`函数在这里进行了Hook和重写，在默认的情况下，将返回`Date.now() - initTime`的值。

如果在开发者工具中打断点，可以看到调用栈如下：

![image 00](image00.png)

可以看到，Unity内模块的各类时间方法最终都会调用到该函数中。

# C#层的两种入口

在C#中，也有两种方式来获取时间戳，分别为`DateTime.Now`和`StopWatch.Start/Stop`方法。

针对`StopWatch`，可以看到，`StopWatch.Start()`和`StopWatch.Stop()`方法都会调用到`performance.now()`方法：
![image 01](image01.png)

但是针对`DateTime.Now`方法，我们却没有触发这个断点，而是触发到了下图所示的断点：
![image 02](image02.png)

在这里并没有调用到`performance.now()`，而是最终调用到了`Date.now()`方法。

# 性能差异对比

经过上述方案，我们了解到在微信小游戏环境下，从C#层获取当前时间一共有两种方案，分别是通过`DateTime`、`StopWatch`，而微信小游戏对性能敏感，因此有必要了解其开销如何。此外，微信小游戏在安卓环境下提供了高精度时间函数，也就是说我们需要关注三种时间戳函数的性能开销。

我们进行重复调用10000次时间戳函数，并且统计占用时间。此外为了进行对比，我们也对一个长度为5000的数组进行冒泡排序，分别对比耗时，相关代码如下：

```cs
    void Start()
    {
        if (datetimeTargetButton != null)
        {
            datetimeTargetButton.onClick.AddListener(() =>
            {
                ToggleDateTime(true); 
                var stopWatch = new System.Diagnostics.Stopwatch();

                var arr = new int[5000];
                for (int i = 0; i < arr.Length; i++)
                {
                    arr[i] = arr.Length - i;
                }

                stopWatch.Start();
                var startTime = DateTime.Now;

                for (int i = 0; i < 100000; i++)
                {
                    var tmpNow = DateTime.Now;
                }

                stopWatch.Stop();

                var tmpNowTime = stopWatch.ElapsedMilliseconds;


                stopWatch.Restart();
                // 完成一次大小为1000的数组的最差排序算法

                // 接着进行排序
                for (int i = 0; i < arr.Length; i++)
                {
                    for (int j = 0; j < arr.Length - 1; j++)
                    {
                        if (arr[j] > arr[j + 1])
                        {
                            var tmp = arr[j];
                            arr[j] = arr[j + 1];
                            arr[j + 1] = tmp;
                        }
                    }
                }
                stopWatch.Stop();

                datetimeTargetText.text = $"DateTime: {tmpNowTime}ms\n, Sort: {stopWatch.ElapsedMilliseconds}ms";
                ToggleDateTime(false); 
            });
        }

        if (stopWatchTargetButton != null)
        {
            stopWatchTargetButton.onClick.AddListener(() =>
            {
                ToggleDateTime(true);
                var stopWatch = new System.Diagnostics.Stopwatch();
                stopWatch.Start();

                for (int i = 0; i < 100000; i++)
                {
                    var tmpNow = System.Diagnostics.Stopwatch.GetTimestamp();
                }

                stopWatch.Stop();
                stopWatchTargetText.text = $"{stopWatch.ElapsedMilliseconds}ms";

                ToggleDateTime(false);
            }); 
        }
    }
```

性能统计结果如下（取10次平均）：

| 平台       | DateTime.Now() | Stopwatch.GetTimestamp() | Sort  |
|------------|----------------|--------------------------|-------|
| 开发者工具 | 19.7ms         | 10.3ms                   | 25.4ms|
| iPhone 14 Pro(高性能模式) |  18.6ms  |  21.2ms  |  24ms  |
| iPhone 14 Pro(普通模式) |  122.3ms  |  41ms |  222.5ms   |
| OPPO Find X| 106.4ms  | 66.1ms  |  73ms |
| OPPO Find X(高精度时间)| 103.1ms  | 109.9ms  |  72.5ms |
---------

目前得出结论如下：

1. 在开发者工具、安卓设备和iOS普通模式下，调用`performance.now()`的性能显著高于`Date.now()`，所用耗时基本上只有后者的50%左右。
2. 高性能模式下则相反，调用`Datetime.Now()`的性能略优于`performance.now()`，原因目前未知。
3. 微信提供的高精度时间函数性能显著低于自带的`performance.now()`函数，因此只建议在Dev Build或者Profiling的时候开启。
