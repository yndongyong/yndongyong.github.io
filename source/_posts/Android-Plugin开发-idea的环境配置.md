---
title: Android Plugin开发 idea的环境配置
categories:
  - - android
  - - android线上填坑
  - - android studio
  - - flutter
  - - compose
tags:
  - - android
date: 2022-11-04 10:07:31
---

### 1 环境配置

插件的build.gradle的配置，一开始各种配置不对，后期研究了文档之后方向，两种可以正常编译运行的方式。

#### 方式一：

```
plugins {
    id 'java'
    id 'org.jetbrains.kotlin.jvm' version '1.6.0'
    id 'org.jetbrains.intellij' version '1.6.0'
}

group 'com.github.Joehaivo'
version '0.1.0-SNAPSHOT'
repositories {
    mavenCentral()
}

dependencies {
    implementation "org.jetbrains.kotlin:kotlin-stdlib:1.6.0"
    testImplementation 'org.junit.jupiter:junit-jupiter-api:5.7.0'
    testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.7.0'

    // https://mvnrepository.com/artifact/org.apache.pdfbox/fontbox
    implementation 'org.apache.pdfbox:fontbox:3.0.0-alpha2'

}

//jar {
//    from configurations.compile.collect { it.isDirectory() ? it : zipTree(it) }
//}

// See https://github.com/JetBrains/gradle-intellij-plugin/
intellij {
//    version = '2020.1'
    version = '2021.2'
//    localPath = project.hasProperty("StudioRunPath") ? StudioRunPath : StudioCompilePath
    type = "IC"
    plugins = ["android"]
}

runIde {
    // Absolute path to installed target 3.5 Android Studio to use as
    // IDE Development Instance (the "Contents" directory is macOS specific):
    ideDir = file("D:\\Android\\android-studio")
}
buildSearchableOptions {
    enabled = false
}

patchPluginXml {
    changeNotes = """
      Add IconFontViewer plugin.<br>
      <em>happy to used</em>"""
    sinceBuild = '191'
    untilBuild = '222.*'
}
test {
    useJUnitPlatform()
}

tasks.withType(JavaCompile) {
    options.encoding = "UTF-8"
}
```

#### 方式二：

```groovy
plugins {
    id 'java'
    id 'org.jetbrains.kotlin.jvm' version '1.6.0'
    id 'org.jetbrains.intellij' version '1.6.0'
}

group 'com.github.Joehaivo'
version '0.1.0-SNAPSHOT'
repositories {
    mavenCentral()
}

dependencies {
    implementation "org.jetbrains.kotlin:kotlin-stdlib:1.6.0"
    testImplementation 'org.junit.jupiter:junit-jupiter-api:5.7.0'
    testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.7.0'

    // https://mvnrepository.com/artifact/org.apache.pdfbox/fontbox
    implementation 'org.apache.pdfbox:fontbox:3.0.0-alpha2'

}

//jar {
//    from configurations.compile.collect { it.isDirectory() ? it : zipTree(it) }
//}

// See https://github.com/JetBrains/gradle-intellij-plugin/
intellij {
    //    version = '2020.1'
    //    version = '2021.2'
    localPath = project.hasProperty("StudioRunPath") ? StudioRunPath : StudioCompilePath
    type = "IC"
    plugins = ["android"]
}

/*runIde {
// Absolute path to installed target 3.5 Android Studio to use as
// IDE Development Instance (the "Contents" directory is macOS specific):
ideDir = file("D:\\Android\\android-studio")
}*/
buildSearchableOptions {
    enabled = false
}

patchPluginXml {
    changeNotes = """
Add IconFontViewer plugin.<br>
<em>happy to used</em>"""
    sinceBuild = '191'
    untilBuild = '222.*'
}
test {
    useJUnitPlatform()
}

tasks.withType(JavaCompile) {
    options.encoding = "UTF-8"
}
```

此外需要注意下：plugin.xml中需要如果有单独是针对Android 平台的相关的配置，是需要加入对Android平台的配置支持的。

```xml
<depends>com.intellij.modules.platform</depends>
<depends>com.intellij.modules.androidstudio</depends>
<depends>org.jetbrains.android</depends>
```

### **注意：**

在前期的过程中，需要如果gradle的使用的版本高于6.6 ，需要配合JDK11，否则编译过程中会报handler_shake xx的错误。是jdk的版本过低，本机就是java8.
要是还报错的话，的要挂梯子了。后面再次搭建环境，遇到了这个报错，通过换了不同的节点处理的，等编译过一遍之后还需要关闭梯子，否则会报其他的错误，不影响编译的问题。

对于引入了第三方的jar的插件，生成的plugin的格式是zip格式，在安装插件的时候要引入zip格式的。如果引入了jar包格式的，会不正常。
插件的生成目录是build/distributions