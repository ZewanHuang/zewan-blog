---
layout:     post
title:      使用 VideoPlayer 在 Unity UI 播放视频
subtitle:   "Use VideoPlayer to play video in Unity UI"
date:       2022-07-15 15:47:39
author:     "Zewan"
no-catalog: true
mathjax:    true
target: _blank
header-style: text
tags:
    - unity
---

在 Unity 项目开发中，有时需要在非全屏 UI 窗口下播放 CG 或其它类型的视频。本文介绍支持多平台播放的 _Video Player_ 组件的用法。

首先，在场景中创建一个 UI 层的 _RawImage_：

![create-rawimage](/img/in-post/post-unity-videoplayer/create-rawimage.png)

接着，在项目资源管理器中创建一个 _RenderTexture_（可命名为 Movie），大小设置为将要播放视频的尺寸：

![create-rt](/img/in-post/post-unity-videoplayer/create-rt.png)

接下来，在场景中创建好的 _RawImage_ 上挂载 _VideoPlayer_ 组件，_Render Mode_ 默认选择 Render Texture，指定视频来源为 URL 并填入视频链接，可以是本地视频地址和在线访问链接。_VideoPlayer_ 能够播放的视频格式为设备内置播放器能够播放的格式。

![videoplayer-settings](/img/in-post/post-unity-videoplayer/videoplayer-settings.png)

在上图也可以看到，添加 _VideoPlayer_ 组件时会默认将音频输出设置好，并自动勾选轨道 Track 0，若不想同时播放视频的音频，取消勾选即可。

最终播放效果图如下：

![player-demo](/img/in-post/post-unity-videoplayer/player-demo.png)