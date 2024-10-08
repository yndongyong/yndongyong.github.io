---
title: Android开发快速定位App顶层的Activity和Fragment
categories:
  - - android
  - - android studio
tags:
  - - android
  - - activity
  - - fragment
date: 2024-08-18 14:33:27
---

# 利用FindActivityFragment插件快速定位Android app顶层Activity和Fragment


## 插件功能介绍 (Plugin Features)
FindActivityFragment插件，可以帮助开发者快速定位并打开正在运行的顶层Activity类和Fragment类，并支持跳转到相应的代码文件。通过简单的快捷键操作，开发者可以立即跳转到相应的代码文件，减少查找的时间。

- 获取当前App的顶层Activity以及其包含的Fragment。
- 打开页面对应的代码类。

![](https://raw.githubusercontent.com/yndongyong/picBed/master/FindActivityFragment.gif)

## 安装和配置 (Installation and Configuration)
可以在JetBrains插件市场中搜索“FindActivityFragment”，然后点击安装。安装完成后，可以在Android Studio的快捷键设置中配置插件的触发快捷键，默认的快捷键"alt 0"，也可以通过 "Code ->FindFindActivity" 菜单

## 使用指南 (How to Use)
使用FindActivityFragment插件非常简单。首先，确保Android Studio工程已经连接到一台正在运行目标应用的设备上。然后使用配置的快捷键，插件将自动获取当前顶层Activity和Fragment，并弹出一个对话框供选择打开的代码类。

## 使用场景 (Use Cases)
对于一个复杂的多页面应用程序，在调试过程中快速定位特定的Activity、Fragment非常重要。FindActivityFragment插件可以帮助开发者快速定位，并立即打开对应的页面代码进行调试。

## 总结 (Conclusion)
该插件是基于Adb命令实现，基本兼容了各个版本的Android studio；分别适配了Android11、Android12、Android13、Android14，Android15。