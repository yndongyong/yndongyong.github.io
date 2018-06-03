---
title: Android异步消息通信小节
date: 2017-07-09 22:05:35
tags: Android,Handler,MessageQueue,Looper
---
试着从整体的角度上去简述Handler、Looper、MessageQueue
三者之间的关系，以及使用这三者构建的Android系统的异步消息通信机制。


三者之间的关系,一图胜千言。

![Handler与Looper与MessageQueue的关系](http://image.lxway.com/upload/1/36/136c06769c6d0a67f1f05d6fc8e504ac.png)

一些术语：

- Message : 线程间通信的消息载体，存储于MessageQueue中。
- MessageQueue : 负责存储Message对象，每个线程只能关联一个MessageQueue。
- Looper :无限消息循环，不停的检索MessageQueue，每个线程只能关联一个Looper。
- Handler：负责消息的分发（将message添加到MessageQueue中）和处理。每个线程可以关联多个Handler。

### 概述 ###

Android 应用初始化的时候，会创建一个主线，通常称为Main Thread,亦或是UI线程。在UI线程的初始化过程中，会创建一个Looper， 同时将此Looper与UI线程绑定，该Looper负责管理应用程序对象包括Activity、以及Activity的window、BroadcastReciver等的通信，一个线程只会有一个Looper实例。

在Looper实例的创建过程中，又会创建一个MessageQueue对象。MessageQueue 负责存储Message，但是Message的添加不是直接通过MessageQueue，而是通过与Looper对象的相关联的Handler。
MessageQueue是消息存储的场所，

Looper无线循环的检索MessageQueue中是否有需要处理的消息，检索到有message时，执行message的target(即Handler)的dispatchMessage方法。

Handler对象在创建时，默认会关联到创建它的当前线程以及当前线程的MessageQueue。默认的线程是不具有Looper以及MessageQueue对象的。使用Handler对象可以将Message和runnable投递到与当前线程相关联的MessageQueue中，同时，可以在message出栈的时候对其进行处理。

### Handler的两个主要作用 ###

-  a. 安排在将来的某个时间点执行message或者runnable。

-  b. 跨线程通信，可以通过其实现子线程和UI线程之间的通信。


### 在子线程更新UI的几种方式 ###

- a. 使用Handler的send or post 系列的方法。
- b. 在Activity 中使用runUiOnThread(runnable)方法
- c. 使用View.post(runnable)方法


### 一些值得注意的地方 ###
- a. 如果想要在自己创建的Thread中使用Looper，可以使用系统提供的HandlerThread类。
- b. 慎用Handler，Handler过长的生命周期会一直持有Activity，造成内存泄露。