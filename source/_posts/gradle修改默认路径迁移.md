---
title: gradle修改默认路径迁移
categories:
  - - android
  - - android线上填坑
  - - android studio
  - - flutter
  - - compose
tags:
  - - android
date: 2022-11-07 14:12:13
---



`gradle`文件夹会越来越膨胀，有全局缓存，依赖库文件、编译缓存。过不来多久c盘就红了，清理了好几次，过不了多久又再次变得很大；而且清理之后再次打开一些不常用的项目时，需要重新下载各种依赖和库，耽搁时间，所以有必要迁移到其他盘上。

1. 将`C:\Users\用户名\.gradle` 的目录复制到目标文件下，比如：`D:\.gradle`
2. 新建环境变量 `GRADLE_USER_HOME`，值为新的.gradle文件夹的路径
3. `android studio` 更新`gradle user home` 的路径为新路径，若没有设置过，`android studio` 会自动以 `GRADLE_USER_HOME`  环境变量指定的为准
4. 重启电脑，如果c盘下的文件迁移不干净，可以支持删除整个文件夹了