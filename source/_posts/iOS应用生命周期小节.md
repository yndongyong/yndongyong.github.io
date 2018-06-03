---
title: iOS应用生命周期小节
date: 2017-12-17 13:13:27
tags: iOS 生命周期
---


### iOS生命周期状态
iOS应用的生命周期状态分为如下几种：

- Not Running （非运行状态） 应用尚未运行或者被系统终止
- Inactive （前台非运行状态） 应用正在进入前台状态，还不能接受事件处理
- Active (前台运行状态) 应用进入前台运行状态，可以接受用户事件处理
- Background （后台运行状态） 应用进入后台，依然能够执行代码。执行完之后将进入挂起状态
- Suspended（挂起状态） 被挂起的应用进入一种“冷冻状态” ，不能执行代码，若系统内存不足，应用会被终止。

![iOS应用的5中状态](http://oav23hfp9.bkt.clouddn.com/17-12-17/35088222.jpg)

在iOS应用的各种状态转换的过程中，iOS系统会回调应用程序的委托对象AppDelegate中的方法，并且能够发送相应的通知。主要的方法和通知如下：

方法                                       | 通知              | 说明
-------------------------------------------|-----------------|--------        
application:didFinishLaunchingWithOptions: | UIApplicationDidFinishLaunchingNotification | 应用启动初始化时会调用该方法并发出通知，这个阶段会实例化根视图控制器
applictaionDidBecomeActive:| UIApplicationDidBecomeActiveNotification| 应用进入前台并且出去活动状态时调用该方法并发出通知，可以在这个方法里恢复UI的状态
applicationWillResignActive：| UIApplicationWillResignActiveNotification | 应用从活动状态进入到非活动状态是调用该方法并发出通知。可以在这个方法里保存UI状态
applicationDidEnterBackground: | UIApplicationDidEnterBackgroundNotification | 应用进入后台时调用该方法并发出通知，可以在这里保存用户数据，释放资源如数据库资源
applicationWillEnterForegound: | UIApplicationWillEnterForegoundNotification | 应用进入到前台，但是处于非活动状态会调用该方法，这个阶段可以恢复用户数据
applicationWillTerminate: | UIApplicationWillTerminateNotification | 应用被终止时调用该方法并发出通知，但是内存清除除外，可以在这里释放资源 或者保存用户数据


iOS应用各种状态切换的方法回调说明见下图
![iOS 生命周期](http://oav23hfp9.bkt.clouddn.com/17-12-17/33791198.jpg)


### 应用启动时的状态切换，对应图中的红线

Not Running -> Inactive -> Active

### 点击home键退出应用时的状态切换，对应图中的绿线
应用不可以后台执行时
Active -> Inactive -> Background -> Suspended -> Not Running

应用可以后台执行时
Active -> Inactive -> Background -> Suspended 

### 挂起从新运行场景，对应图中的蓝线
Suspended -> Background -> Inactive -> Active 

### 应用挂起后被清除时不会调用任何回调方法
清除包括系统内存不足的系统清除 和 用户从多任务列表中清除

注意：

- 应用是否可以在后台运行和挂起，由info.plist的属性Application dose not run in bakground属性决定，对应的键是UIApplicationExitsonSuspended。
