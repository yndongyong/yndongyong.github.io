---
title: Android Apk大小优化
categories:
  - - android
tags:
  - - android
date: 2022-04-19 16:37:58

---

## 背景		

随着App工程的长时间维护，项目里堆积了一些曾经有用的、但随着升级迭代逐渐没有使用到的资源，比如`Activity` 、`Fragment`、`RecyclerView`的`ItemView`等的布局`xml`资源，各种背景`Shape`的`xml`资源，以及图标图片等。虽然在`Release`包的构建配置中设置了`shrinkResources true`的配置,但并没有将这些资源真正的删除，而是替换为一个个的空文件，仍然会参与编译构建，最终导致。

长此以往，造成了三个主要问题：

1. 工程代码包括资源的膨胀。
2. `Apk`的包大小越来越大的问题，此前v5.5.0版本的armv8a的包大小已经达到`69M`。进行包大小优化有助于提高下载转化率。
3. 编译耗时增加，目前的工程本地一次完整的`debug`包的全量编译需要耗时七八分钟之久（编译环境win11,corei5-1135g7，24g内存，m.2 ssd）。从编译速度的角度考虑，无用资源的移除也是很有必要的。

## 优化方法

为了处理上述的三个问题，优化`Apk`包大小，主要采用以下三种手段

1.  `android lint` 移除无用资源
2.  移除`drawable-xhpi`图片资源文件夹中的无用图片资源
3.  使用图片压缩工具将`drawable-xxhpi`图片资源文件夹中的尺寸比较的图片资源进行压缩减小图片大小。

### 一、`android lint` 移除无用资源

在`Android studio`可以通过lint检查分析那些没有使用到的资源，操作路径为： `Analyze`->`Run INspection by Name` 输入`unused resources`对分析出的资源文件加以清理。

如果直接删除`lint`分析出来的文件列表，可能会存在误删除的情况，比如`raw`文件中的文件，和通过反射方式以及`getResources().getIdentifier`的方法使用的资源`lint`是无法区分的，需要我们提前进行梳理，使用白名单进行保留，可以新建一个resource的xml文件，列出要保留的文件，防止误删。

````xml
<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:keep="@drawable/ic_reoad_event_*,@drawable/ic_reoad_marker_*,@drawable/ic_reoad_marker_big_*,
    @dimen/status_bar_height,@dimen/navigation_bar_height,@raw/bg_video_login,@raw/roll,@string/MI_APP_ID,@string/MI_APP_KEY"
    >
</resources>
````

在移除资源的过程需要各个小项仔细区分，防止避免误删除。建议整个工程都在git的版本管理之下再执行该操作。

此外，在实际执行`Remove All Unused Resource`的过程中还出现了删除各种布局的控件id的情况，也需要避免，我们通过在`App moudle`的lint.xml文件中，定制扫描规则进行忽略。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<lint>
    <issue id="SpUsage" severity="ignore" />

    <issue id="SmallSp" severity="ignore" />

    <issue id="ContentDescription" severity="ignore"/>

    <!-- 加上可以避免布局中的控件id被删除 -->
    <issue id="UnusedIds" severity="ignore"/>

</lint>
```



这种`lint`方式也存在一定的缺陷，它是一个静态扫描工具，并没有考虑到`ProGuard`工程中删除的无用代码，`lint`检查不出这些无用代码所引用的无用资源。



### 二、`drawable-xhpi`中的无用资源移除

对于`drawable-xhpi`的优化，我在App（目前的月活大概是数十万量级）中进行埋点，经过两个月的数据沉淀，通过分析埋点数据发现98%的设备的屏幕分辨率都是`1080*1920`以及之上的，还有1%的720P。基于以上的埋点数据来看`drawable-xhpi`的文件夹已经没有存在的必要了，但是为了防止之前导入图片只放一份在`drawable-xhpi`的情况，需要对比`drawable-xhpi`文件夹和`drawable-xxhpi`，将那些存在与`xxhdpi`也存在与`xhpi`的图片从`drawable-xxhpi`文件中删除，防止误删除情况的发生。

使用了`python`编写了一个简单的脚本达到上诉目的

```python
# -*- coding: UTF-8 -*-
import os
import shutil

top = os.getcwd()
print 'scan top: ' + top
xDpi = os.path.join(top,"drawable-xhdpi")
print 'scan xDpi dir: ' + xDpi

xxDpi = os.path.join(top,"drawable-xxhdpi")
print 'scan xxDpi dir: ' + xxDpi

xxList = []
xList = []

for path in os.listdir(xxDpi):
	apps = os.path.join(xxDpi, path)
	xxList.append(path)
print 'xxList: '
print xxList 


for path in os.listdir(xDpi):
	apps = os.path.join(xDpi, path)
	xList.append(path)
print 'xList: '
print xList 

a = [x for x in xList if x in xxList]
print '删除xh中与xxh中的相同元素: '
print len(a)
print a

for path in a:
	filePath = os.path.join(xDpi, path)
	os.remove(filePath)
```

此外，依据埋点数据的结论，在团队中推行，图片只用放一份到`xxhdpi`文件夹中的开发规范。

### 三、大图压缩

对于剩余的`drawable-xxhpi`中的图片资源，使用一款名叫鸭鸭压缩软件进行批量压缩，至于压缩效果前期已经找到UI老师进行确认。



## 优化效果

1. 移除`drawable-xhpi`中的无用图片资源820份
2. 移除布局 135份 ；移除`shape drawable` 131； 移除`drawable-xxhpi`未使用的图片400多份，以及各类`string`、`color`资源500多份
3. v5.5.0版本的armv8a的包大小从`69.5M`包大小降低至`63.08M`，包大小减少`6.42M`

![附图基于v5.5.0版本进行优化的包大小对比](https://raw.githubusercontent.com/yndongyong/picBed/master/img/202204211417724.png?token=ACWZC7FJXPTKPHNL6QD6WS3CMD3TY)
