---
title: App minSdk升级为24之后的问题
categories:
  - - android
  - - android线上填坑
tags:
  - - android minSdk
date: 2022-10-08 11:35:01
---

### 升级的原因

最近，因为vivo商店的审核问题，将App的`minSdk`从21升级到24，下限支持为Android7。`ci`上打包的`apk`，发给测试之后，测试反馈打开之后就闪退。

### 出现的问题

排查日志，有一个`SIGE（11）`相关的日志，使用未加固的包安装测试，发现正常。因而认为是加固引起的问题。此前的加固策略里有一条是签名检测，加固包签名检测到不一致时，会退出App。

### 原因

这里是`build.gradle`中签名相关的配置

```groovy
signingConfigs {
    release {
        storeFile file("../xxx.jks")
        storePassword "xxxx"
        keyAlias "xxxx"
        keyPassword "xxxx"
    }
}
```

怀疑大概率是签名机制的问题，`Android7`及以上默认是使用v2签名。

`minSdk`是21时：包构建时使用v1签名打包，然后加固，最后重新使用v1+v2签名。

`minSdk`升级为24之后：包构建时使用v2签名打包，然后加固，最后重新使用v1+v2签名。

两种行为之下，在加固的签名校验环节任务签名不一致了，主动关闭App了。

### 解决

`minSdk24`的情况下，构建包时，配置使用使用`v1`进行签名，`v2`签名需要关闭（默认是打开的），加固之后使用`v1+v2`方式重新签名

配置如下：

```groovy
signingConfigs {
    release {
        storeFile file("../xxx.jks")
        storePassword "xxxx"
        keyAlias "xxxx"
        keyPassword "xxxx"
        v2SigningEnabled false
        v1SigningEnabled true
    }
}
```

### 出现新问题

打个某个页面会出现闪退，sentry上也有关于改崩溃的大量记录

```java
java.lang.UnsatisfiedLinkError: couldn't find DSO to load: libgifimage.so caused by: Dynamic section string-table not found result: 0
    at com.facebook.soloader.SoLoader.doLoadLibraryBySoName(TbsSdkJava:825)
    at com.facebook.soloader.SoLoader.loadLibraryBySoName(TbsSdkJava:673)
    at com.facebook.soloader.SoLoader.loadLibrary(TbsSdkJava:611)
    at com.facebook.soloader.SoLoader.loadLibrary(TbsSdkJava:559)
    at com.facebook.soloader.NativeLoaderToSoLoaderDelegate.loadLibrary(TbsSdkJava:25)
    at com.facebook.soloader.nativeloader.NativeLoader.loadLibrary(TbsSdkJava:44)
    at com.facebook.animated.gif.GifImage.ensure(TbsSdkJava:42)
    at com.facebook.animated.gif.GifImage.createFromNativeMemory(TbsSdkJava:88)
    at com.facebook.animated.gif.GifImage.decodeFromNativeMemory(TbsSdkJava:110)
    at com.facebook.imagepipeline.animated.factory.AnimatedImageFactoryImpl.decodeGif(TbsSdkJava:88)
    at com.facebook.fresco.animation.factory.AnimatedFactoryV2Impl$1.decode(TbsSdkJava:88)
    at com.facebook.imagepipeline.decoder.DefaultImageDecoder.decodeGif(TbsSdkJava:139)
    at com.facebook.imagepipeline.decoder.DefaultImageDecoder$1.decode(TbsSdkJava:60)
    at com.facebook.imagepipeline.decoder.DefaultImageDecoder.decode(TbsSdkJava:120)
    at com.facebook.imagepipeline.producers.DecodeProducer$ProgressiveDecoder.doDecode(TbsSdkJava:316)
    at com.facebook.imagepipeline.producers.DecodeProducer$ProgressiveDecoder.access$300(TbsSdkJava:136)
    at com.facebook.imagepipeline.producers.DecodeProducer$ProgressiveDecoder$1.run(TbsSdkJava:186)
    at com.facebook.imagepipeline.producers.JobScheduler.doJob(TbsSdkJava:224)
    at com.facebook.imagepipeline.producers.JobScheduler.access$000(TbsSdkJava:24)
    at com.facebook.imagepipeline.producers.JobScheduler$1.run(TbsSdkJava:90)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
    at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(TbsSdkJava:50)
    at java.lang.Thread.run(Thread.java:919)
```

发现是页面上有个gif动态图，使用的fresco-git先关的库展示的，`libgifimage.so`等相关的so库是有`fresco`的引入的。`fresco`有自己的一套`soloader`机制，加固之后so加载不到了。

最终在加固策略中排除`fresco`相关的一系列so库，不对这部分so库加固，就能解决问题。

相关so库有

- `libnative-imagetranscoder.so`
- `libimagepipeline.so`
- `libnative-filters.so`
- `libgifimage.so`