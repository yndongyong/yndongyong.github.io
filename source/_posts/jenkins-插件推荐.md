---
title: jenkins 插件推荐
categories:
  - - android
  - - jenkins
tags:
  - - android
  - - jenkins
date: 2022-08-20 14:16:38
---

记录一下有用的插件，方便以后重装时了解必装的插件。

### build-user-vars-plugin:1.9 

会在环境变量里，注入启动job的用户的相关的信息，方便获取构建者，支持BUILD_USER_ID，BUILD_USER_NAME等信息。

pipeline使用方式：

```groovy
wrap([$class: 'BuildUser']) {
    script {
        BUILD_USER = "${env.BUILD_USER}"
    }
}
```

貌似最新的版本里，可以直接通过env.BUILD_USER拿到了。

### build-name-setter:2.2.0 

设置构建名称,pipeline step中设置

```groovy
buildName "#${BUILD_NUMBER}-${params.BUILD_TYPE_SELECT}-${env.BUILD_USER}"
```



### git-parameter:0.9.17 

支持git 的参数化构建，分支，tag等过滤选择。

pipeline parameters

```groovy
gitParameter( branchFilter: 'origin/release/.*', defaultValue: "origin/release/v$__VERSION_NAME", name: 'BUILD_GIT_BRANCH', type: 'PT_BRANCH', selectedValue: 'TOP', sortMode: 'DESCENDING_SMART', useRepository: "$__GIT_REPO")
```



### envinject:2.875.v9b_9e962da_a_ec 

环境变量参数注入

### thinBackup:1.10 

用于jenkins备份,相关配置

![thinBackup备份设置](https://raw.githubusercontent.com/yndongyong/picBed/master/img/20230304111117.png)

回复时，需要执行该操作，让jenkins重新读取数据

![重新读取配置数据](https://raw.githubusercontent.com/yndongyong/picBed/master/img/20230304111229.png)

注意： 备份恢复之后，因为是文件的所有者是root，jenkins的docker的用户是jenkins，访问恢复的文件就存在权限问题

解决方案,给目录的拥有者uid 1000：sudo chown -R 1000 _data ，还是设置了一下文件夹的权限（非必需）。

### versionnumber:1.9 

方便设置构建的时候的版本号。支持有BUILD_OF_WEEK 、BUILD_OF_DAY等信息

```groovy
__versionNumberOfToday = VersionNumber(versionNumberString: '${BUILD_DATE_FORMATTED,"yyyy-MM-dd"}.${BUILDS_TODAY}',worstResultForIncrement: 'SUCCESS');
```



### rebuild:1.34

方便使用同样的参数，进行再次构建。这个很有用，在上一次构建失败的情况或者多次执行相同参数构建的情况下。

### pipeline-utility-steps:2.13.0 

pipeline下提供一些有用的方法，比如fileExit、readJson、cleanWs等方法。