---
title: 小米手机安装magisk获取root权限配置抓包证书
categories:
  - - android
tags:
  - - 刷机
  - - root
  - - magisk
date: 2023-02-11 14:59:14
---

之前手机已经刷机为pixelexperience 系统，为了root权限，折腾一下magisk。

# 1.参考

- [Download Magisk Manager Latest Version 25.2 For Android 2022](https://magiskmanager.com/) magisk官方介绍



# 2.magisk安装步骤：

1. 解锁bootloader，之前刷机时已经解锁了，解锁参考[红米note 8 pro 刷pixelexperience | yndongyong‘s blog](https://yndongyong.github.io/2023/03/03/红米note-8-pro-刷pixelexperience/)

2. 从rom包提取boot.img，手机连接电脑 ，拷贝到手机sd卡根目录。

   ```
   adb push boot.img /sdcard
   ```

   3.安装Magisk app，打开App，安装->下一步->选择并修补一个文件,从手机sd卡根目录选择boot.img 。“开始”生成补丁img

4. 修补成功，会在sd卡根目录生成一个**magisk_patched-版本_随机.img**

5. 进入fastboot模式，

   ```sh
   adb reboot bootloader
   ```

6. 刷入magisk生成的img

```sh
fastboot flash boot magisk_patcerd_xxx.img
```

7. 出现fish字样就是刷好了，然后重启进入手机，打开magisk app，会显示magsik的版本。	

# 3.Magisk隐藏root

## 1.magisk随机包名隐藏

magisk app 设置 –> 隐藏Magisk应用 –>  允许来自此来源的应用（开启）–>  输入新应用名称 –>成功修改包名 –>继续安装。

## 2.magisk隐藏root

1. 安装shamiko模块，[Shamiko v0.6](https://github.com/LSPosed/LSPosed.github.io/releases/tag/shamiko-126)
2. magisk 右上角设置  -> 打开 Zygisk选项->打开`配置排除列表` ->选择需要隐藏root的app，需要把app下所有模块都选上。
3. 重启手机。

# 4.抓包软件证书配置

 1. 安装各种抓包软件（charles 、httpcanary）对应的证书，此时是安装在系统目录下。

    用户目录

```shell
/data/misc/user/0/cacerts-added/
```

​		系统目录

```shell
/system/etc/security/cacerts
```

2. 将用户证书移动到系统目录下

   ```shell
   #申请root权限
   su root
   #将系统文件系统重新挂载为可读写
   mount -o remount,rw /
   #进入用户证书目录
   cd /data/misc/user/0/cacerts-added
   #移动证书到系统目录
   mv * /etc/security/cacerts/ 
   #将权限改为只读
   mount -o remount,ro /
   ```

3. 某些app采用固定证书时，还需要安装JustTrueMe模块。

   - 先安装LSPosed模块https://hub.fgit.ml/LSPosed/LSPosed/releases/latest
   - 在安装JustTrustMe模块[Releases · SekiBetu/JustTrustMe (github.com)](https://github.com/SekiBetu/JustTrustMe/releases)