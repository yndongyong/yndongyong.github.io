---
title: Android12升级适配
categories:
  - - android
tags:
  - - android
date: 2023-02-20 15:19:02
---

最近在做Android12 的升级适配

主要改动的点，app/build.gradle文件中需要将targetsdk 、compileSdkVersion都指定到31（Android12）

- 对外暴露的组件需要export为true
- 有105个语法错误，主要是intent.getString的返回值是空的，语法更严格了。
- 微信sdk需要适配，否则微信登录分享不可用，主要是软件包可见性的问题。
- 软件包可见性，否则打不开对应的app，微信，qq、云闪付。涉及范围：登录、分享、支付。
- 定位权限，分为粗略定位和精确定位，请求精确定位时必需一起请求粗略定位，经排查正常
- 蓝牙权限和定位权限分离，新的三个蓝牙权限适配。这里需要明确区分android12以上还是一下，动态事情不同的权限。
- pendintent适配,在 Adnroid 12 之前，默认创建一个 PendingIntent 它是可变的。所以这里指定成还是可变的避免引起问题。影响通知，应用内更新功能，桌面快捷方式功能。
- 修复android12上没有read_phone_state权限时获取获取连接类型奔溃的问题。模拟器上跑 java.lang.RuntimeException: Unable to create application com.tengyun.yyn.system.TravelApplication: java.lang.SecurityException: getDataNetworkTypeForSubscriber 没有readphonestate权限
- 修复获取网络连接性在原生系统上总是网络不可用的问题，原来对于网络的连通性判断还处理需要认证的网络情况，但是在类原生系统上，网络连通性是访问google服务器，国内连不上，总是判断为不可用。
- 播放器sdk播放时内部会获取getNetworkTypeForSubscriber时没有read_phone_state权限奔溃的问题。

