---
layout:     post
title:      如何在 Unity Editor 中录制游戏界面
subtitle:   "How to Record Game Screen in Unity Editor"
date:       2022-07-14 15:47:39
author:     "Zewan"
catalog:    true
mathjax:    true
target: _blank
header-style: text
tags:
    - Unity
    - 图形学CG
---

在 Unity Editor 中录屏的方式主要有仅限 Windows 平台的 Unity 自带录屏和官方录屏插件 _Unity Recorder_，它们共有的功能有：

* 自定义输出视频的分辨率，不受限于屏幕的分辨率
* 支持输出多种类型的输出，如视频、动画片段、序列帧、GIF、全景视频等
* 效果较佳的视频图片压缩

与 Unity 自带录屏相比，插件 _Unity Recorder_ 有以下更多优点：

* 不仅限于 Windows 平台
* 能够同时录制多个机位，即多个 Camera 镜头的输出
* 能够与 Timeline 共同使用

当然，这两种方法**仅用于编辑器中**，无法在构建 OS、Android、WebGL 等项目中使用。

下面分别介绍这两种方式的使用过程。

## 自带录屏

Unity Editor 自带录屏功能，仅在 Windows 平台中使用，否则菜单栏不出现录制选项。

首先，如下图所示，点击菜单栏 Window，依次点击 General > Recorder > RecorderWindows。

![built-in-recorder-window](/img/in-post/post-unity-recorder/built-in-recorder-window.png)

选中完毕之后，将弹出 Recorder 窗口，随后点击 Add New Recorders，可以选择录制的内容，例如动画片段、视频、图片序列、GIF 等。

![built-in-recorder-export-types](/img/in-post/post-unity-recorder/built-in-recorder-export-types.png)

在录制视频的窗口中，可以自己设置帧率、格式、分辨率、视频输出路径等。设置完毕后，点击红色按钮，将自动运行项目和录制 Game 界面。

![built-in-recorder-export-setting](/img/in-post/post-unity-recorder/built-in-recorder-export-setting.png)

## Unity Recorder 插件

_Unity Recorder_ 插件有更多的功能，以下介绍插件的安装和使用。

### 插件导入

点击菜单栏的 _Window_ 内的 _Package Manager_，打开包管理器，切换至 _Packages:Unity Registry_，如下图所示。

![pkg-manager](/img/in-post/post-unity-recorder/pkg-manager.png)

在管理器右上角搜索框中，输入 _Unity Recorder_ 找到该插件，点击 _Install_。

### 使用 TimeLine 录屏

首先，在资源中创建 TimeLine 对象：

![create-timeline](/img/in-post/post-unity-recorder/create-timeline.png)

随后，在 TimeLine 中左侧加号按钮，点击添加 Recorder Track，创建之后在右侧添加 Recorder Clip：

![create-track](/img/in-post/post-unity-recorder/create-track.png)

![create-clip](/img/in-post/post-unity-recorder/create-clip.png)

双击 Clip，在 _Inspector_ 面板中，对其进行配置：

![edit-clip](/img/in-post/post-unity-recorder/edit-clip.png)

详细配置如下：

* Selected recorder：选择录制类型，如视频 Movie 等
* Capture：设置录制屏幕对象，如 Game 界面或者某个相机的捕获界面
* Format：选择输出视频格式
* Output File：设置视频保存路径

配置好后，在场景 Scene 中的某个对象上建立 _Playable Director_ 组件，将刚刚创建的 _TimeLime_ 对象拖拽到 Playable 位置中。

![recorder-plugin-set-object](/img/in-post/post-unity-recorder/recorder-plugin-set-object.png)

此处勾选 _Play On Awake_ 表示运行游戏时自动启动录制，当 _TimeLine_ 时间指针走完 _Recorder Clip_ 片段时，将自动保存到之前配置的文件夹中。

若不勾选，则需要在代码中控制其启动。_Playable Director_ 组件在命名空间 `UnityEngine.Playables` 中，当编写代码控制该选项时，需要先引用命名空间，然后代码获得挂载在 Scene 中的 _Playable Director_ 组件对象，使用 `Play()` 和 `Stop()` 函数进行操控。

---

至此，_Unity Editor_ 中两种录屏方式全部介绍完毕。

> 本来想多介绍可用于 iOS 等平台的录屏插件 [NatCorder](https://assetstore.unity.com/packages/tools/integration/natcorder-video-recording-api-102645)，奈何现在已经已下架，新的跨平台录屏插件又需要 money，所以就没介绍了。