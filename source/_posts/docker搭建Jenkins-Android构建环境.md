---
title: docker搭建Jenkins+Android构建环境
categories:
  - - linux
tags:
  - - android
  - - ci
date: 2023-02-15 17:29:18
---

通过docker file定制化jenkins容器，基于jenkins:lts-jdk11长期版本。

jenkins构建Android，需要JDK、android sdk、gradle。jenkins:lts中已经包含了jdk11，其余都已经安装并注入了环境变量

**需要注意的几个点：**

- `jenkins-plugin-cli` 需要连接jenkins plugin.io，可能会下载plugin失败，jenkins启动之后需要检查一下对应的插件是否安装。
- **jenkins已安装plugin**： rebuild:1.34 gradle-daemon:0.1.0 gradle:2.3.1 build-name-setter:2.2.0 git-parameter:0.9.18 envinject:2.901.v0038b_6471582 thinBackup:1.15 pipeline-utility-steps:2.15.1 versionnumber:1.10 build-user-vars-plugin:1.9
- androidsdk 已经安装了sdkmanager cli，并同意了license，jenkins 会自动下载需要的sdk版本。
- flutter：已经配置了国内镜像

**环境变量**

- **JAVA_HOME**= /opt/java/openjdk 
- **ANDROID_HOME**=/var/jenkins_home/android/sdk
- **FLUTTER_HOME**=/var/jenkins_home/flutter
- **GRADLE_USER_HOME**=/var/jenkins_home/.gradle



# 1.通过dockerfile构建镜像

1.创建Dockerfile文件

```sh
mkdir deploy_android
cd deploy_android
touch Dockerfile
```

2.Dockerfile内容，依据需求改动

```shell
FROM jenkins/jenkins:lts-jdk11
LABEL maintainer yndongyong@foxmail.com

USER root

ENV TZ=Asia/Shanghai 
  
# 设置变量 JAVA_HOME 已经在path中
ENV JAVA_HOME="/opt/java/openjdk"
ENV ANDROID_HOME="/var/jenkins_home/android/sdk" \
     SDK_TOOL_URL="https://dl.google.com/android/repository/commandlinetools-linux-8512546_latest.zip" \
     COS_CLI_URL="https://cosbrowser-1253960454.cos.ap-shanghai.myqcloud.com/software/coscli/coscli-linux" \
     FLUTTER_HOME="/var/jenkins_home/flutter"\
     FLUTTER_RELEASE_URL="https://storage.flutter-io.cn/flutter_infra_release/releases/stable/linux/flutter_linux_3.3.10-stable.tar.xz"

# 创建android sdk目录,并下载 sdkmanager
RUN mkdir -p $ANDROID_HOME \
     && cd $ANDROID_HOME \
     && curl -o sdk.zip $SDK_TOOL_URL \
     && unzip sdk.zip \
     && rm sdk.zip \
     && cd cmdline-tools \
     && mkdir latest \
     && mv bin/ latest \
     && mv lib/ latest \
     && mv NOTICE.txt latest \
     && mv source.properties latest 

# 安装android sdk其他package, 输入y是因为此处会有一个licence,需要用户同意后才会安装 
RUN echo y | ${ANDROID_HOME}/cmdline-tools/latest/bin/sdkmanager  --licenses \
    && echo y | ${ANDROID_HOME}/cmdline-tools/latest/bin/sdkmanager  "platforms;android-29" "build-tools;29.0.3"

RUN chmod -R a+w $ANDROID_HOME  \
    && chown -R jenkins $ANDROID_HOME

  
#更换一个软件源
#安装flutter需要的插件
RUN sed -i -e "s/deb.debian.org/mirrors.tuna.tsinghua.edu.cn/" /etc/apt/sources.list.d/debian.sources && \ 
    apt-get update && \
    apt-get install -y \
       xz-utils \
      &&  apt-get clean

    
# 设置环境变量: 把 android sdk 路径加入到 PATH 中
ENV PATH="$PATH:$ANDROID_HOME:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools:$FLUTTER_HOME/bin:$FLUTTER_HOME/bin/cache/dart-sdk/bin"



#flutter
RUN echo "Flutter sdk" && \
    cd /var/jenkins_home && \
    curl -o flutter.tar.xz $FLUTTER_RELEASE_URL && \
    tar xf flutter.tar.xz && \
    git config --global --add safe.directory $FLUTTER_HOME && \
    flutter config --no-analytics && \ 
    rm -f flutter.tar.xz && \ 
    chmod -R 775 flutter  
    
# Flutter 设定镜像
ENV PUB_HOSTED_URL=https://pub.flutter-io.cn
ENV FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn

#创建gradle_cache目录，由job指定项目的gradle缓存目录，否则cleanworkspace之后，缓存不在了,/var/jenkins_home被映射到host了，永久存在
ENV GRADLE_USER_HOME="/var/jenkins_home/.gradle"

RUN mkdir -p $GRADLE_USER_HOME \
     && cd $GRADLE_USER_HOME \
     && chown -hR jenkins $GRADLE_USER_HOME \
     && chmod -R a+w $GRADLE_USER_HOME 
    
#将gradle全局属性拷贝到GRADLE_USER_HOME
#COPY ./init.gradle $GRADLE_USER_HOME
#COPY ./gradle.properties $GRADLE_USER_HOME


#将walle-cli等拷贝
COPY ./lib/ /sdk 

#处理cos
# .cos.yaml 拷贝到 /sdk/cos #检查cos
COPY ./.cos.yaml /sdk/cos/
RUN cd /sdk/cos \
     && curl -o coscli-linux $COS_CLI_URL \
     && mv coscli-linux coscli \
     && chown -R jenkins /sdk/cos \
     && chmod 755 coscli \
     && /sdk/cos/coscli ls cos://app-pkg-1254950508 -c /sdk/cos/.cos.yaml
#里面有一些jar包
#RUN chmod -R 775 /sdk

#使用内嵌的cli安装一些插件 
RUN jenkins-plugin-cli --plugins rebuild:1.34 gradle-daemon:0.1.0 gradle:2.3.1 build-name-setter:2.2.0 git-parameter:0.9.18 envinject:2.901.v0038b_6471582 thinBackup:1.15 pipeline-utility-steps:2.15.1 versionnumber:1.10 build-user-vars-plugin:1.9

USER jenkins

```



# 2.配置docker-compose.yml

```shell
version: "3.4"
services:
  jenkins:
    build:
      context: .
      dockerfile: Dockerfile
    image: yndongyong/jenkins-android:v0.0.1
    container_name: jenkins
    restart: always
    environment:
      JAVA_OPTS: "-server -Djava.awt.headless=true -Duser.timezone=Asia/Shanghai"
    ports:
      - "80:8080"
    volumes:
      - "jenkins-data:/var/jenkins_home"
      - "./jenkins_backup:/var/jenkins_backup"
#      - "./lib:/sdk"
#    deploy:
#        resources:
#            limits:
#              cpus: "8"
#              memory: 16G
#            reservations:
#              cpus: "0.25"
#              memory: 4g  
volumes:
  jenkins-data:
    external: true

##需要特别注意jenkins_backup的owner是docker用户，docker jenkins_home是Jenkins，需要权限设置
##sudo chown -R 1000:1000 jenkins_backup
```

这里使用了一个docker volumes "jenkins-data".

可以提前创建好

```shell
docker volume create --name=jenkins-data
```

同时还映射了一个host的jenkins_backup目录，thinBackup插件指定的备份数据存放到`/var/jenkins_backup`目录。方便备份数据拷贝存放，但是要注意目录权限问题

```sh
sudo chown -R 1000:1000 jenkins_backup
```



# 3.启动容器

```sh
docker compose up -d
```
