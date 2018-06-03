---
title: Activity生命周期小节
date: 2017-07-05 22:03:42
tags: [Android,Activity,生命周期]
---
Activity被设计用来与用户进行交互，界面元素的展示、交互动效、业务逻辑等都将在这里依赖业务逻辑以及其间的关系进行编码。
Activity是整个应用生命周期的重要组成部分。所有的activity都会被 Activity Stack进行管理，当一个新的Activity被启动，它将会被系统放入Activity Stack 并且处于栈顶，而之前的Activity依然停留在Activity Stack中，处于新的Activity之下。

### Activity的几种状态 ###

- running  Activity对用户可见，处于屏幕的最前，即Activity Stack的栈顶
- paused  当前Activity被dialog和或者非全屏的Activity覆盖，此时，Activity对于用户可能可见，且不再具有与用户交互的能力。
- stopped 另一个Activity来到了前台，当前的Activity完全被覆盖对用户不可见。
- destroyed Activity处于 paused或者stopped状态时被系统回收了。


  

### Activity向开发者提供了一整个完整的生命周期hook函数 ###

 ![Activity生命周期图](http://ww1.sinaimg.cn/mw690/a0b1fa45gy1fha7psnvraj20e90ifmz4.jpg)

onCreate(bundle) -> onStart() -> onResume() -> onPause() ->onStop() -> onDestroy()

可见的生命周期：onStart() 与onStop() 两个函数调用之前的这段时间，Activity对于用户是可见的。

前台生命周期：onResume() 与 onPause() 两个函数发生调用的这段时间内，Activity是处于**前台**的。

- onCreate() 所有的Activity对需要实现onCreate(bundle)方法，在这里为window 设置ui，绑定数据。

- onStart ()  Activity 即将进入前台被用户可见。

- onResume()  Activiy进入前台且能够与用户交互。

- onPause()  系统开始调用另一个Activity的生命周期；可以在在这里进行数据的保存，持久化，停止动画，释放CPU资源等的轻量级操作。**只有当这个方法放回之后系统才会调用另一个Activity的生命周期**，所以请不要在该方法里执行耗时的操作，以避免造成UI卡顿。

- onStop()  Activity 不在被用户可见，新的Activity已经处于running状态，可以在这里了持久化一些数据，释放内存资源。

- onDestroy() Activity 进入了destroyed的状态 ，被调用了finish方法或者被系统回收内存而被清理。

- onRestart() Activity 进入stopped 状态时，重新被带入前台。

  在实现这些生命周期hook函数时，**总是需要调用父类的方法**。

  ​

### 各种情况下Activity生命周期的调用顺序###

- 当Activity被启动时各个生命周期方法的调用顺序

   ![启动Activity](http://oav23hfp9.bkt.clouddn.com/act%E8%A2%AB%E6%89%93%E5%BC%80%E6%97%B6%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.png)

-  当Activity处于running状态按下返回键时

   ![按下返回键](http://oav23hfp9.bkt.clouddn.com/17-7-6/act%E6%8C%89%E8%BF%94%E5%9B%9E%E9%94%AE.png)

-  当Activity处于running状态按下Home键时

   ![按下Home键时](http://oav23hfp9.bkt.clouddn.com/17-7-6/act%E6%8C%89%E4%B8%8Bhome%E9%94%AE.png)

- 从多任务切换器切换时

   ![](http://oav23hfp9.bkt.clouddn.com/17-7-6/%E6%8C%89%E4%B8%8BHome%E9%94%AE%EF%BC%8C%E4%BB%8E%E5%A4%9A%E4%BB%BB%E5%8A%A1%E5%88%87%E6%8D%A2%E5%9B%9E.png)

-  当Activity处于running状态切换到另一个Activity时

   ![](http://oav23hfp9.bkt.clouddn.com/17-7-6/%E5%88%87%E6%8D%A2%E5%88%B0%E5%8F%A6%E4%B8%80%E4%B8%AAact.png)

-  从另一个Activity切换回时

   ![](http://oav23hfp9.bkt.clouddn.com/17-7-6/%E4%BB%8E%E5%8F%A6%E5%A4%96%E4%B8%80%E4%B8%AA%E5%88%87%E5%9B%9E%E6%97%B6.png)

- 按下锁屏键
  
  ![按下Home键时](http://oav23hfp9.bkt.clouddn.com/17-7-6/act%E6%8C%89%E4%B8%8Bhome%E9%94%AE.png)
	
- 解锁时

![](http://oav23hfp9.bkt.clouddn.com/17-7-6/%E6%8C%89%E4%B8%8BHome%E9%94%AE%EF%BC%8C%E4%BB%8E%E5%A4%9A%E4%BB%BB%E5%8A%A1%E5%88%87%E6%8D%A2%E5%9B%9E.png)

   

### Configuration changes 对生命周期的影响 ###

设备的configuration 发生改变时，都会最先通知到Activity处，系统会将Activity执行destroy操作，然后重新激活Activity，会重走一遍生命周期。

在新的系统版本里，由竖屏切换到横屏，或者横屏切换到竖屏都只会重走一遍生命周期，这个时候空间的一些状态会被保存，重走生命周期的时候状态会被恢复，但是用户的输入等数据不会被系统保存，需要开发者自己处理。

有些时候需要我们绕过重走生命周期的流程，Manifest文件中，为Activity配置属性：android:configChanges;可以为该属性指定很多种值，已达到当这些值对应的情况改变打的时候，不重走生命周期，而是回调到Activity的onConfigurationChanged方法。

以下配置可以达到**横竖屏切换时，不重启Activity**的效果：

 ```java
 android:configChanges="orientation|keyboardHidden|screenSize"
 ```


