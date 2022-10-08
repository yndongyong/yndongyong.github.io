---
title: Android端代码分支管理规范
categories:
  - android
date: 2022-03-29 09:43:08
tags:
- 分支管理
---

#### 背景

目前，App的发版呈现发版频率增加的趋势，以及面临各种紧急需求临时发版或者临时修复线上严重问题导致频繁发版的情况。总体来说是发版频率增加了，由原本的一月一次（或者一月两次）提升到多次的情况，现在的代码分支管理策略已经不能适应目前项目的规模和发版节奏相匹配 了。

原来的分支管理，是按照App迭代版本号创建与迭代同名的主分支加上若干特性分支方式。，例如，规划了下个迭代发布2.0.0版本，那么就基于线上版本对应的v1.0.0分支创建一个v2.0.0的主分支（用于线上bug修复），以及若干的其他feature分支，等到开发完成需要提测时，会将确认要提测的所有的feature分支合并到的v2.0.0主分支，后续提测问题的修复都将v2.0.0分支进行，并最终基于这个分支进行打包上线。 这种方式适合版本维护比较单一，迭代周期固定的项目。

![1](https://raw.githubusercontent.com/yndongyong/picBed/master/img202203291110534.png?token=GHSAT0AAAAAABRNWEVXKAJLADMHSQ3RXBBQYSHDSWQ)

在频繁发版的情况下，这种模式存在以下几个比较难以解决的的问题。

- 问题1，在临近上线前，由于某个特性功能由于延期或者其他原因不需要上线了， 这个时候噩梦开始，在主干分支上剥离某个特新分支的代码是非常困难，git的日志上会存在大量剔除代码的记录，同时由于对应的特性feature分支上有没有bug修复记录，难以继续在该分支上维护。尤其在多版本并行开发的时候，这个问题会更加突出。
- 问题2， 在需要临时发版2.0.0版本上线其他需求时， 分支v2.0.0已经被占用，只能取其他的名字，这样的情况次数多了以后，会造成版本记录混乱，功能点不明确。

#### **新的方案**

参考阿里**AoneFlow**方案，这种方案下使用四种分支类型， 1个master主干分支+1个hotfix分支+N个特性分支+N个发布分支 

**规则一：** 开始工作前，从master创建1个hotfix分支与N个feature分支。
从代表最新已发布版本的master分支上，创建hotfix分支修改线上版本问题，创建以**feature/** 前缀命名的若干特性分支，进行特性功能的开发和以及提测问题的修复，每个功能模块对应一个特性分支。

![2](https://raw.githubusercontent.com/yndongyong/picBed/master/img202203291112562.png?token=GHSAT0AAAAAABRNWEVXNBBTAVSXK3UZEQ34YSHDTXQ)

**规则二：** 通过合并feature分支，形成release分支。
从master分支上拉出以**release/**前缀命名的新分支，将本次要集成或者发布的feature、hotfix分支依次合并过去。

![3](https://raw.githubusercontent.com/yndongyong/picBed/master/img202203291113966.png?token=GHSAT0AAAAAABRNWEVWDSMS3J6NJOF547AAYSHDUDA)

优势：

1. 多个特性分支可同步开发
2. 发布分支的特性是动态组成的，调整起来是非常容易的。要上线哪些feature不上线哪些feature，重新分布分支只用将原来的发布分支删除掉，从主干分支拉出新的同名分支，再将需要上线的分支合并过来，很好的避免了原来方案的弊端。

**规则三：**  使用release分支发布到线上正式环境后，合并相应的release分支到master分支，在master分支上添加tag，同时删除该release分支关联的feature分支。

![4](https://raw.githubusercontent.com/yndongyong/picBed/master/img202203291114069.png?token=GHSAT0AAAAAABRNWEVWLTCATX25LZREFRW6YSHDULA)

 版本发布后需要将这条发布分支合并到主干分支，主干分支的最新版本始终与线上版本一致，如果需要回溯历史版本，只需要在主干分支上找到对应的版本标签。
为了避免在代码仓库里堆积大量历史特性分支，还应该清理掉已经上线历史的部分特性分支 。

#### 工具链

 每个分支的创建 简单合并步骤使用单纯的 Git 命令就能玩转，但还是将这些常规化操作工具化了，避免重复这些日常琐事的命令操作。
目前在Jenkins提供了两个Job，

- YYN_CREATE_BRANCHS:用于操作hotfix,feature分支创建（目前删除远程分支的操作还是交由团队个人）
- YYN_MERGE_BRANCHS:拉release发布分支以及合并hotfix、feature分支Job。

#### **其他约定**

- master分支开发人员不允许直接提交代码，已经设置分支保护。
- 定期清理历史release发布分支。
