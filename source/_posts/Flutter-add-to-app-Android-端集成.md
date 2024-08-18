---
title: Flutter add-to-app Android 端集成
categories:
  - - android
  - - flutter
tags:
  - - android
date: 2023-08-21 15:24:57
---

记录一下在现有Android端项目集成Flutter工作中遇到的问题。

flutter版本3.3.10 如下：

![flutter环境](https://raw.githubusercontent.com/yndongyong/picBed/master/img/Snipaste_2023-01-05_17-14-49.png)



集成的主要步骤参考官方的文档[将 Flutter module 集成到 Android 项目 - Flutter 中文文档 - Flutter 中文开发者网站 - Flutter](https://flutter.cn/docs/development/add-to-app/android/project-setup)

因为现有的原生工程已经迭代很久，拥有完善的网络框架：支持网关切换、请求加解密、通用的请求参数，业务接口异常埋点等功能，埋点组件、路由组件等很多基础业务组件，这部分的能力希望复用，而不是在flutter上再次实现一遍，如果这样，前期flutter的引入需要很大的代价去实现一系列基础组件,与我们目前引入flutter的初衷：降本增效——降低开发成本，提升开发效率相违背。同时如果存在两套组件，比如网络组件，后期也会增加维护成本。需要尽可能多的复用原生已有的能力。

flutter 的集成有如下两种方式

- 源码依赖集成
- `aar`集成

### 集成目标

​	一：在开发阶段如果如果不涉及flutter相关的开发、以及打包的时候则使用`aar`集成的方式对flutter无感，避免隐形的编译成本。

​	二：涉及flutter相关的开发则切换的源码依赖的方式，毕竟要使用原生宿主的能力，需要先将app跑起来，在使用`flutter attach`进行开发调试。

需要有一种很简单的方式，可能动态的改变flutter组件是aar依赖，还是源码依赖，这一块复用我们之前在组件化过程的配置的脚本即可。

### 源码依赖集成

第一步，在`settings.gradle`文件中配置，将flutter工程引入到宿主工程中

```groovy
def flutterEmbedFile = file('../xxx_fluter/.android/include_flutter.groovy')   // new
if (flutterEmbedFile.exists() && debugable) {
    setBinding(new Binding([gradle: this]))                                          // new
    evaluate(file(                                                               // new
            '../xxx_fluter/.android/include_flutter.groovy'                         // new
    ))
}
```

第二步，在`app/build.gradle`中引入flutter模块 

```groovy
dependencies {
  implementation project(':flutter')
}
```

之后就可以在app工程中使用`io.flutter.embedding.android.FlutterActivity`

**注意**：源码依赖集成时最好使用git submodel的方式，单独管理flutter工程

### aar依赖集成

#### aar包构建

目前`flutter 3.0`中已经提供了完善的构建aar的工具了，不会再有之前的构建的aar包中没有包含上第三方依赖的问题了。构建flutter aar产物的命令是 `flutter build aar` 会在本地生成`maven repo`，不方便远程依赖，而我们希望是能够直接将aar产物上传到公司的maven私库中。

使用 `flutter build aar -v`命令，详细的了解的该命令的执行过程，发现就是执行了sdk目录中的 `aar_init_script.gradle` 脚本。目录为 `FlutterSdk\flutter\packages\flutter_tools\gradle`

```groovy
 executing: [D:\code_space\xxx_fluter\.android/]  D:\code_space\xxx_fluter\.android\gradlew.bat -I=D:\FlutterSdk\flutter\packages\flutter_tools\gradle\aar_init_script.gradle
-Pflutter-root=D:\FlutterSdk\flutter -Poutput-dir=D:\code_space\xxx_fluter\build\host -Pis-plugin=false -PbuildNumber=1.0 --full-stacktrace --info -Pverbose=true -Ptarget=lib\main.dart
-Pdart-obfuscation=false -Ptrack-widget-creation=true -Ptree-shake-icons=false -Ptarget-platform=android-arm,android-arm64,android-x64 assembleAarDebug

```

该脚本的大概作用就是给`gradle project`配置上`maven-publish plugin`，同时进行相关的配置

基本该脚本，修改了一份 `flutter_aar_upload.gradle`，支持配置publish到远程仓库。改动的地方可搜索 `add by dong`查看

```groovy
// This script is used to initialize the build in a module or plugin project.
// During this phase, the script applies the Maven plugin and configures the
// destination of the local repository.
// The local repository will contain the AAR and POM files.

import java.nio.file.Paths
import org.gradle.api.Project
import org.gradle.api.artifacts.Configuration
import org.gradle.api.publish.maven.MavenPublication

void configureProject(Project project, String mavenUrl ,String mavenUser ,String mavenPwd) {
    if (!project.hasProperty("android")) {
        throw new GradleException("Android property not found.")
    }
    if (!project.android.hasProperty("libraryVariants")) {
        throw new GradleException("Can't generate AAR on a non Android library project.");
    }

    // Snapshot versions include the timestamp in the artifact name.
    // Therefore, remove the snapshot part, so new runs of `flutter build aar` overrides existing artifacts.
    // This version isn't relevant in Flutter since the pub version is used
    // to resolve dependencies.
    project.version = project.version.replace("-SNAPSHOT", "")

    if (project.hasProperty("buildNumber")) {
        project.version = project.property("buildNumber")
    }

    project.android.libraryVariants.all { variant ->
        addAarTask(project, variant)
    }

    project.publishing {
        repositories {
            if (mavenUrl.startsWith("file://")){
                maven {
                    name = "local"
                    //默认本地路径
                    url = uri("${mavenUrl}/outputs/repo")
                }
            }else if (mavenUrl.startsWith("http")){
                maven {
                    name = "remote"
                    //指定要上传的maven私服仓库
                    url = mavenUrl
                    allowInsecureProtocol = true
                    //认证用户和密码
                    credentials {
                        username mavenUser
                        password mavenPwd
                    }
                }
            }
        }
    }

    // Some extra work to have alternative publications with the same format as the old maven plugin.
    // Instead of using classifiers for the variants, the old maven plugin appended `_{variant}` to the artifactId

    // First, create a default MavenPublication for each variant (except "all" since that is used to publish artifacts in the new way)
    project.components.forEach { component ->
        if (component.name != "all") {
            project.publishing.publications.create(component.name, MavenPublication) {
                from component
            }
        }
    }

    // then, rename the artifactId to include the variant and make sure to remove any classifier
    // data tha gradle has set, as well as adding a <relocation> tag pointing to the new coordinates
    project.publishing.publications.forEach { pub ->
        def relocationArtifactId = pub.artifactId
        pub.artifactId = "${relocationArtifactId}_${pub.name}"
        pub.alias = true

        pub.pom.distributionManagement {
            relocation {
                // New artifact coordinates
                groupId = "${pub.groupId}"
                artifactId = "${relocationArtifactId}"
                version = "${pub.version}"
                message = "Use classifiers rather than _variant for new publish plugin"
            }
        }

    }

    // also publish the artifacts in the new way, using one set of coordinates with classifiers
    project.publishing.publications.create("all", MavenPublication) {
        from project.components.all
        alias false
    }

    if (!project.property("is-plugin").toBoolean()) {
        return
    }

    String storageUrl = System.getenv('FLUTTER_STORAGE_BASE_URL') ?: "https://storage.googleapis.com"
    // This is a Flutter plugin project. Plugin projects don't apply the Flutter Gradle plugin,
    // as a result, add the dependency on the embedding.
    project.repositories {
        maven {
            url "$storageUrl/download.flutter.io"
        }
    }
    String engineVersion = Paths.get(getFlutterRoot(project), "bin", "internal", "engine.version")
            .toFile().text.trim()
    project.dependencies {
        // Add the embedding dependency.
        compileOnly ("io.flutter:flutter_embedding_release:1.0.0-$engineVersion") {
            // We only need to expose io.flutter.plugin.*
            // No need for the embedding transitive dependencies.
            transitive = false
        }
    }
}
//add by dong 5
void configurePlugin(Project project, String mavenUrl ,String mavenUser ,String mavenPwd) {
    if (!project.hasProperty("android")) {
        // A plugin doesn't support the Android platform when this property isn't defined in the plugin.
        return
    }
    configureProject(project, mavenUrl,mavenUser,mavenPwd)
}

String getFlutterRoot(Project project) {
    if (!project.hasProperty("flutter-root")) {
        throw new GradleException("The `-Pflutter-root` flag must be specified.")
    }
    return project.property("flutter-root")
}

void addAarTask(Project project, variant) {
    String variantName = variant.name.capitalize()
    String taskName = "assembleAar$variantName"
    project.tasks.create(name: taskName) {
        // This check is required to be able to configure the archives before `publish` runs.
        if (!project.gradle.startParameter.taskNames.contains(taskName)) {
            return
        }
        // Generate the Maven artifacts.
        finalizedBy "publish"
    }
}

// maven-publish has to be applied _before_ the project gets evaluated, but some of the code in
// `configureProject` requires the project to be evaluated. Apply the maven plugin to all projects, but
// only configure it if it matches the conditions in `projectsEvaluated`

allprojects {
    apply plugin: "maven-publish"
}

projectsEvaluated {
    assert rootProject.hasProperty("is-plugin")
    if (rootProject.property("is-plugin").toBoolean()) {
        assert rootProject.hasProperty("maven-url")
        // In plugin projects, the root project is the plugin.
        configureProject(rootProject,  rootProject.property("maven-url"),
                rootProject.property("maven-user"), rootProject.property("maven-pwd"))
        return
    }
    // The module project is the `:flutter` subproject.
    Project moduleProject = rootProject.subprojects.find { it.name == "flutter" }
    assert moduleProject != null
    //add by dong 1
    println("moduleProject: " + moduleProject)

    assert moduleProject.hasProperty("maven-url")
    configureProject(moduleProject, moduleProject.property("maven-url"),
            moduleProject.property("maven-user"), moduleProject.property("maven-pwd"))

    // Gets the plugin subprojects.
    Set<Project> modulePlugins = rootProject.subprojects.findAll {
        it.name != "flutter" && it.name != "app"
    }
    // When a module is built as a Maven artifacts, plugins must also be built this way
    // because the module POM's file will include a dependency on the plugin Maven artifact.
    // This is due to the Android Gradle Plugin expecting all library subprojects to be published
    // as Maven artifacts.
    modulePlugins.each { pluginProject ->
        configurePlugin(pluginProject, moduleProject.property("maven-url"),moduleProject.property("maven-user"), moduleProject.property("maven-pwd"))
        moduleProject.android.libraryVariants.all { variant ->
            // Configure the `assembleAar<variantName>` task for each plugin's projects and make
            // the module's equivalent task depend on the plugin's task.
            String variantName = variant.name.capitalize()
            moduleProject.tasks.findByPath("assembleAar$variantName")
                    .dependsOn(pluginProject.tasks.findByPath("assembleAar$variantName"))
        }
    }
    //add by dong 4
    String mavenUrl = moduleProject.property("maven-url")
    String version = moduleProject.property("buildNumber")

    println("Version: $version")
    println("MavenUrl: " + mavenUrl)

    //输出 配置
    String buildMode = moduleProject.gradle.startParameter
            .taskNames.find { it.startsWith("assembleAar") }.substring(11)
    println("BuildMode: $buildMode")
}

//获取Flutter Engine Version
def flutterEngineVersion(Project project) {
//    Process process = Runtime.getRuntime().exec(["sh", "-c", "which flutter"] as String[])
//    BufferedReader reader = new BufferedReader(new InputStreamReader(process.inputStream))
//    process.waitFor()
//    String path = reader.readLine()
    String path = getFlutterRoot(project)
    println("Flutter Path：" + path)
//    String engineVersion = Paths.get(path, "internal", "engine.version")
//            .toFile().text.trim()
    String engineVersion = Paths.get(path, "bin", "internal", "engine.version")
            .toFile().text.trim()
    println("Flutter Engine Version: " + engineVersion)
    return engineVersion
}


```

同时，将调用该文件的脚本以及需要传递的参数放置于脚本 `build_flutter_aar.sh` 中,方便构建产物。

```shell
#!/usr/bin/env sh
# 从script目录放回上一层级
cd ..
# 自动生成.android文件夹
flutter pub get
cd .android

# 参数说明
# Pflutter-root flutter sdk的目录
# -Pmaven-url 可以是本地路径，也可为远程地址
# 示例：
# maven-url=file://D:/code_space/xxx_fluter/build/host
# maven-url=http://xxxxxxxxxxx/repository/ynty_android/
# maven-user 与 maven-pwd 为maven远程用户及密码，若发布到本地无需设置
# -PbuildNumber=1.0.0 为版本号
# 最后 assembleAarRelease 为打包类型，assembleAarDebug, assembleAarProfile

#debug包 
./gradlew \
 -I=../script/flutter_aar_upload.gradle \
 -Pflutter-root=D:/FlutterSdk/flutter \
 -Pmaven-url='http://xxxxxxx/repository/android/' \
 -Pmaven-user=xxxx \
 -Pmaven-pwd=xxxxx \
 -Pis-plugin=false \
 -PbuildNumber='1.0.0' \
 --full-stacktrace --info -Pverbose=true -Ptarget='lib\main.dart' -Pdart-obfuscation=false \
 -Ptrack-widget-creation=true \
 -Ptree-shake-icons=false -Ptarget-platform='android-arm,android-arm64,android-x64' assembleAarDebug

#release包
./gradlew \
 -I=../script/flutter_aar_upload.gradle \
 -Pflutter-root=D:/FlutterSdk/flutter \
 -Pmaven-url='http://xxxxx/repository/android/' \
 -Pmaven-user=xxxx \
 -Pmaven-pwd=xxxx \
 -Pis-plugin=false \
 -PbuildNumber='1.0.0' \
 --full-stacktrace --info -Pverbose=true -Ptarget='lib\main.dart' -Pdart-obfuscation=false \
 -Ptrack-widget-creation=true \
 -Ptree-shake-icons=false -Ptarget-platform='android-arm,android-arm64,android-x64' assembleAarRelease

```

#### app工程flutter产物aar集成

1. app中配置**maven**仓库地址

   ```groovy
    String storageUrl = System.getenv('FLUTTER_STORAGE_BASE_URL') ?: "https://storage.googleapis.com"
   	#flutter 国内镜像 storageUrl = https://storage.flutter-io.cn
   repositories {
       maven {
       	url '私库地址xxxx'
       }
       maven {
       	url 'https://$storageUrl/download.flutter.io'
       }
   }
   ```

2. app/build.gradle中引入

   ```groovy
   dependencies {
       def flutter_aar_sdk_version = "1.0.0"
       debugImplementation "$gourp_id:flutter_debug:$flutter_aar_sdk_version"
       releaseImplementation "$groupd_id:flutter_release:$flutter_aar_sdk_version"
      //flutter engine version 不用直接依赖了
             
   }
   ```

### CI 自动化构建

jenkins 新建job,执行`build_flutter_aar.sh`负责构建flutter的aar产物，使用aar包构建的脚本自动上传到maven私库，aar的版本号使用对应的git库的master分支最后一次提交的commit。

