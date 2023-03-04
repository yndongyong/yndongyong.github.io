---
title: aosp编译环境搭建以及构建镜像
categories:
  - - android
  - - framework
tags:
  - - android
date: 2023-02-06 15:58:50
---

# 1. 参考文档

- [Cuttlefish 虚拟安卓设备  |  Android 开源项目  |  Android Open Source Project](https://source.android.com/docs/setup/create/cuttlefish?hl=zh-cn)  官网对cuttlefish的介绍以及搭建
- [墨鱼：重启和重置  |  Android 开源项目  |  Android Open Source Project](https://source.android.com/docs/setup/create/cuttlefish-restart?hl=zh-cn) cutterfish相关操作命令
- [Android 开发者代码实验室  |  Android 开源项目  |  Android Open Source Project](https://source.android.com/docs/setup/start?hl=zh-cn) 官网队aosp环境搭建介绍
- [下载源代码  |  Android 开源项目  |  Android Open Source Project](https://source.android.com/docs/setup/download/downloading?hl=zh-cn) 官网源码下载的介绍

# 2.硬件要求

- 需要 64 位环境linux
- 硬盘容量至少500G，代码检出（aosp-master分支：350G左右）与编译（150G左右）均需要大量硬盘空间，~~最后综合考虑检出的是android10分支的代码，硬盘占用少（120G多），编译耗时也会相应减少。~~后面发现使用cuttlefish模拟器在Android11/12上才开始退出，又再次下载了Android12的r26分支，编译使用cuttlefish启动不了编译好的镜像，在android issue网站上看到队intel 11/12th的cpu支持有问题，已经在master分支上修复，最终检出aosp-master分支，有从头开始编译，这又是另外一个悲伤的故事。
- 内存16G起步，越大越好，本机为24G（实际可用23G），在不配置swap的情况下，多次爆内存，表现为terminal闪退。

> 从 2021 年 6 月起，Google 使用 72 核机器，内置 RAM 为 64 GB，完整构建过程大约需要 40 分钟（增量构建只需几分钟时间，具体取决于修改了哪些文件）。相比之下，RAM 数量相近的 6 核机器执行完整构建过程需要 3 个小时。

开发机时间数据：在cpu为Intel-i5 1135G（4核心8线程），24G内存+8G Swap，编译android13,过程大约持续4个小时。编译Android10大约耗时：3个半小时（3个小时20多分）

# 3.软件要求

- 操作系统Ubuntu20.4，如果需要使用cuttlefish模拟器，在该版本下可以避免很多依赖问题。编译aosp可用在Ubuntu18.04、Ubuntu22版本下进行，经过实验都可以正常编译成功，但是cuttlefish在这些版本下的依赖有很多问题，cuttlefish的部署debhelper-compat的12以及以上，但在Ubuntu18下，debhelper-compat为11，无法升级。在Ubuntu22下，descripte依赖版本又太高，降级的话还把在桌面环境搞坏掉，因此强烈推荐在Ubuntu20下搭建编译环境。

- 验证 KVM 可用性,返回一个数值。

  ```shell
  grep -c -w "vmx\|svm" /proc/cpuinfo
  ```

- 编译所需要的依赖

  `sudo apt-get install git-core gnupg flex bison build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 libncurses5 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig`

  aosp源码内包含了用于构建的jdk、python版本了。

  

- swap配置，物理内存不足有爆内存的可能。

  - 创建swap文件,这里设置为8G

  ```bash
  #停用 SWAP
  sudo swapoff -v /swapfile
  #8g
  sudo dd if=/dev/zero of=/swapfile bs=1024M count=8
  #设置读写权限
  sudo chmod 600 /swapfile
  #标注swap区域
  sudo mkswap /swapfile
  #激活swap分区
  sudo swapon /swapfile
  #查看swap是否工作
  sudo swapon --show
  sudo free -h
  #将创建的 SWAP 分区设置为永久分区（开机自启）,将 SWAP 路径写入到/etc/fstab文件中
  /swapfile swap swap defaults 0 0
  
  #其他命令
  #停用 SWAP
  sudo swapoff -v /swapfile
  #删除
  sudo rm /swapfile
  ```

  实践看下来swap只占用了一般。

# 4.安装 Cuttlefish

## 4.1在终端窗口中，下载、构建和安装主机 Debian 软件包：

```shell
sudo apt install -y git devscripts config-package-dev debhelper-compat golang curl
git clone https://github.com/google/android-cuttlefish
cd android-cuttlefish
for dir in base frontend; do
  cd $dir
  debuild -i -us -uc -b -d
  cd ..
done
sudo dpkg -i ./cuttlefish-base_*_*64.deb || sudo apt-get install -f
sudo dpkg -i ./cuttlefish-user_*_*64.deb || sudo apt-get install -f
sudo usermod -aG kvm,cvdnetwork,render $USER
sudo reboot
```

# 5.编译aosp

## 5.1安装repo

Android源码aosp包含了数百个git库，为了管理好这些git库，google开发repo工具，是一系列pythone脚本对git的封装，用来简化管理aosp的源码。

首要先安装git curl，python ubuntu20已近自带了，没有的话也需要安装

```shell
sudo apt-get install git curl
```

值得注意的地方Ubuntu20安装的是python3，需要将python链接到python3，repo环境下有影响。

`sudo ln -s /usr/bin/python3 /usr/bin/python`

接下来创建bin，并加入到PATH中。

```shell
mkdir ~/bin
PATH=~/bin:$PATH
```

下载repo并设置权限，这里使用的清华的镜像

```shell
curl https://mirrors.tuna.tsinghua.edu.cn/git/git-repo > ~/bin/repo
chmod a+x ~/bin/repo
```

## 5.2 下载源码

建立工作目录

```
mkdir android_aosp
cd android_aosp
```

指定repo tuna的源用于更新自己，~/.zshrc里添加：

指定repo更新 使用tuna的源 

```sh
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo/'
```

初始化仓库，可以通过-b指定版本,例如  -b android-12.1.0_r26

```sh
repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest
```

至于要下载哪些分支的源码，分支列表参见	[代号、标记和 build 号  |  Android 开源项目  |  Android Open Source Project](https://source.android.com/docs/setup/about/build-numbers?hl=zh-cn#source-code-tags-and-builds)

同步源码:

```sh
repo sync
```

需要经过漫长的等等。晚上估计需要下载3,4个小时。

# 6.编译镜像

使用 `envsetup.sh` 脚本初始化环境：

```sh
source build/envsetup.sh
```

使用lunch 选择要构建的目标,因为要使用cutterfish模拟器，所以使用_cf相关的，

```sh
lunch aosp_cf_x86_64_phone-userdebug
```

userdebug代表构建类型，一共有三种构建类型，

| 构建类型  | 使用情况                                                     |
| :-------- | :----------------------------------------------------------- |
| user      | 权限受限；适用于生产环境                                     |
| userdebug | 与“user”类似，但具有 root 权限和调试功能；是进行调试时的首选编译类型 |
| eng       | 具有额外调试工具的开发配置                                   |

构建代码

```sh
m
```

构建过程依据机器配置的不同，需要几个小时的时间。

构建产物生成目录

`out/producte/`

## 6.1相关构建命令

**m** - 从树的顶部运行构建系统，全量编译。

**mma** - 构建当前目录中的所有模块及其依赖项。

**mmma** - 构建提供的目录中的所有模块及其依赖项。

**croot** - `cd` 到树顶部。

**clean** - `m clean` 会删除此配置的所有输出和中间文件.

# 7. 使用cutterfish启动aosp构建

```
launch_cvd --daemon --start_webrtc=true
```

访问**https://localhost:8443**与虚拟设备交互

## 7.1cutterfisn先关命令

**launch_cvd** - 启动设备

**powerwash_cvd** -将一个 Cuttlefish 设备重置为其初始磁盘状态

**stop_cvd** - 执行不正常关机并停止设备

**adb reboot** - 使设备完成完整的关机过程，将更改同步到磁盘并确保进程关闭

**restart_cvd** -立即关闭 Cuttlefish 设备执行不正常关机，相当于将电池断开并重新连接到物理设