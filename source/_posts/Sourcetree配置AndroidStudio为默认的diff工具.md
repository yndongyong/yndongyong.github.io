---
title: Sourcetree配置AndroidStudio为默认的diff工具
categories:
  - - android
tags:
  - - android studio
date: 2023-05-12 16:16:10
---

配置操作路径Tools -> Options -> External Diff/Merge 都选自定义

### diff配置

Diff Command: xxxpath\studio64.exe 
Arguments: diff $LOCAL $PWD/$REMOTE

### merge配置

Merge Command: xxxpath\studio64.exe 
Arguments: merge $LOCAL $PWD/$REMOTE $PWD/$BASE $MERGED