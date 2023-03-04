---
title: 红米note 8 pro 刷pixelexperience
categories:
  - - android
tags:
  - - android
date: 2023-02-10 17:43:22
---



这款手机官方的rom版本已经永久的停留在Android11，近期app的targetsdk已经升级到3（android12），没有一个android12 的手机属实不方便，便开始了刷机的折腾之路。

**pixelexperience** 刷机包下载地址：[https://get.pixelexperience.org/begonia](https://get.pixelexperience.org/begonia)
**pixelexperience twrp下载地址：**[https://get.pixelexperience.org/begonia](https://get.pixelexperience.org/begonia)

**驱动安装**
刷机前请确保已近安装对应的驱动，或者通过安装小米手机助手PC安装驱动。
需先确保开发者选项开启，usb调试开启，本机有adb工具环境

**bootloader解锁**
参考官网解锁文档[https://www.miui.com/unlock/index.html](https://www.miui.com/unlock/index.html)
需要注册小米账号，绑定手机号，申请解锁（首次的申请的需要等待好久审核），申请通过之后在使用官方提供的解锁工具进行解锁，具体步骤：
1.进入“设置 -> 开发者选项 -> 设备解锁状态”中绑定账号和设备；
2.手动进入Bootloader模式（关机后，同时按住开机键和音量下键）；
3.通过USB连接手机，在解锁软件中点击 “解锁”按钮；



**刷入第三方recovery**

1. pixelexperience 提供了专用的的twrp，下载地址：[https://get.pixelexperience.org/begonia](https://get.pixelexperience.org/begonia)

2. 连接手机至电脑，关机状态下长按音量减+电源键进入fastboot模式，亦可通过下列命令进入

   `adb reboot bootloader`

3. 当手机进入fastboot模式，使用如下命令验证，正常的话会识别出可用设备，没有输出内容的话，大概率是驱动问题，可以去windows设备管理下看是否有android设备，处理驱动问题，小米提供的驱动文件在win11上安装不了，或者直接右键选择更新驱动设备，选择通用串口设备下的驱动。

   `fastboot devices`

4. 刷入recovery

   `fastboot flash recovery <recovery_filename>.img`

5. 当刷入recovery完成时，直接安装音量加+电源键，进入recovery模式，这一步千万不能直接进入系统，否者recovery又会被替换为官方的。

**刷入rom**

1. 进入recovery后，清除数据 Factory Reset -> Format data / factory reset

2. 返回主界面，通过“Apply Update”-> “Apply from ADB” 刷入rom，执行以下命令：

   `adb sideload rom_name.zip`

3. 刷好之后，返回主界面选择 “Reboot system now ” 重启系统。