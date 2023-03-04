---
title: Adnroid gradle plugin 升级到7问题记录
categories:
  - - android
  - - android线上填坑
  - - android studio
  - - flutter
  - - compose
tags:
  - - android
date: 2022-12-11 11:18:54
---

问题1

```groovy
FAILURE: Build failed with an exception.

* What went wrong:
A problem occurred configuring root project 'xxxxxx'.
> Could not resolve all dependencies for configuration ':classpath'.
   > Using insecure protocols with repositories, without explicit opt-in, is unsupported. Switch Maven repository 'maven2(http://xxxxx/repository/xxxxandroid/)' to redirect to a secure protocol (like HTTPS) or allow insecure protocols. See https://docs.gradle.org/7.3.3/dsl/org.gradle.api.artifacts.repositories.UrlArtifactRepository.html#org.gradle.api.artifacts.repositories.UrlArtifactRepository:allowInsecureProtocol for more details. 

* Try:
> Run with --info or --debug option to get more log output.
> Run with --scan to get full insights.

```

配置变更

```groovy
pluginManagement{
    repositories {
        maven {
            url "file://$rootDir/${local_mvn}"
        }
        maven {
            url "${ty_mvn}"
            credentials {
                username xxx_name
                password xxx_pwd
            }
            allowInsecureProtocol = true
        }
		// 华为 maven 仓库地址
        maven {
            url 'http://developer.huawei.com/repo/'
            allowInsecureProtocol = true
        }    
        maven { url "${ali_public}" }
        maven { url "${ali_google}" }
        google()
        gradlePluginPortal()
        mavenCentral()
    }
}
```

加入的该语句 `allowInsecureProtocol=true`

