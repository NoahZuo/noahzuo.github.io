---
title: 怎样本地运行Unity的WebGL包？
tags: 
 - Unity
 - WebGL
 - 微信小游戏
category: Unity
date: 2024-09-23 17:50:33
description: 这篇文章记录了怎样在本地运行Unity的Web包。
---

## 背景

Unity可以进行Web平台的打包，但是打出来的包（入口为index.html）通常无法直接在浏览器中运行，必须使用本地服务器进行反向代理才能正常运行。

正常来说，我们可以使用nginx等工具进行手动的反向代理操作，从而在本地运行打出来的Web包。

只是这种方案费时费力——而Unity本身提供了`Build And Run`的功能，内置了反向代理，本文将介绍怎样使用Unity自带的反向代理功能，直接运行构建出来的Web包。

## SimpleWebServer
在Unity自带的`WebGLSupport`模块中，有一个`SimpleWebServer.exe`程序。顾名思义，这就是一个简单的本地网络服务器，这也就是Unity的`Build And Run`中反向代理所使用到的程序。

我们尝试在GrepWin中搜索`SimpleWebServer`字段，成功定位到`UnityEditor.WebGL.Extensions.dll`：

![GrepWin](grepwin.png)

那么，ILspy启动，定位到程序集为`Unity.Automation.Players.WebGL`：

![ILSpy](ILSpy.png)

可以看到，SimpleWebServer提供Start和Stop函数，因此我们只需要写一个界面，控制`SimpleWebServer`的开始和结束即可。

## 界面及代码

我实现了一个简单的界面如下：

![Tool](Tool.png)

代码也已提供，直接复制粘贴即可：

```cs
using UnityEditor;
using UnityEngine;
using System.IO;
using System;
using System.Net;
using System.Net.Sockets;
using Unity.Automation.Players.WebGL;

public class WebGLDeployTool : EditorWindow
{
    private string webglBuildPath = "";
    private int port = 8080;
    private bool isRunning = false;
    private string errorMessage = "";

    [MenuItem("Tools/WebGL本地部署")]
    public static void ShowWindow()
    {
        GetWindow<WebGLDeployTool>("WebGL本地部署");
    }

    private void OnGUI()
    {
        GUILayout.Label("WebGL本地部署工具", EditorStyles.boldLabel);

        EditorGUILayout.BeginHorizontal();
        webglBuildPath = EditorGUILayout.TextField("WebGL构建路径", webglBuildPath);
        if (GUILayout.Button("浏览...", GUILayout.Width(80)))
        {
            webglBuildPath = EditorUtility.OpenFolderPanel("选择WebGL构建目录", "", "");
        }
        EditorGUILayout.EndHorizontal();

        port = EditorGUILayout.IntField("服务器端口", port);

        EditorGUILayout.BeginHorizontal();
        if (GUILayout.Button("启动服务器"))
        {
            StartServer();
        }
        if (GUILayout.Button("停止服务器"))
        {
            StopServer();
        }
        EditorGUILayout.EndHorizontal();

        if (isRunning)
        {
            EditorGUILayout.HelpBox($"服务器正在运行: http://localhost:{port}", MessageType.Info);
        }
        else
        {
            EditorGUILayout.HelpBox("服务器未运行", MessageType.Warning);
        }

        if (!string.IsNullOrEmpty(errorMessage))
        {
            EditorGUILayout.HelpBox(errorMessage, MessageType.Error);
        }
    }

    private void StartServer()
    {
        errorMessage = "";

        if (string.IsNullOrEmpty(webglBuildPath))
        {
            errorMessage = "请先选择WebGL构建目录";
            return;
        }

        if (!Directory.Exists(webglBuildPath))
        {
            errorMessage = "指定的目录不存在";
            return;
        }

        if (IsPortInUse(port))
        {
            errorMessage = $"端口 {port} 已被占用，请更换端口";
            return;
        }

        StopServer();

        try
        {
            UnityEngine.Debug.Log($"尝试启动服务器: 目录={webglBuildPath}, 端口={port}");
            SimpleWebServer.Start(webglBuildPath, $"http://localhost:{port}/");
            isRunning = true;
            UnityEngine.Debug.Log($"服务器启动成功: http://localhost:{port}");
        }
        catch (Exception e)
        {
            errorMessage = $"启动失败: {e.Message}";
            UnityEngine.Debug.LogError($"服务器启动异常: {e}");
        }
    }

    private void StopServer()
    {
        try
        {
            if (isRunning)
            {
                SimpleWebServer.Stop();
                isRunning = false;
                UnityEngine.Debug.Log($"服务器已停止: http://localhost:{port}");
            }
        }
        catch (Exception e)
        {
            errorMessage = $"停止失败: {e.Message}";
            UnityEngine.Debug.LogError($"服务器停止异常: {e}");
        }
    }

    private bool IsPortInUse(int port)
    {
        try
        {
            using (var tcp = new TcpClient(AddressFamily.InterNetwork))
            {
                tcp.Connect(IPAddress.Loopback, port);
                return true;
            }
        }
        catch (SocketException)
        {
            return false;
        }
    }

    private void OnDestroy()
    {
        StopServer();
    }
}
```