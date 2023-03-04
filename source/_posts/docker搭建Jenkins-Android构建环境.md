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

jenkins构建Android，需要JDK、android sdk、gradle。jenkins:lts中已经包含了jdk11。

# 1.通过dockerfile构建镜像

1.创建Dockerfile文件

```sh
mkdir deploy_android
cd deploy_android
touch Dockerfile
```

2.Dockerfile内容，依据需求改动

```shell
#基于jenkins镜像
FROM jenkins/jenkins:lts-jdk11
USER root
#设置时区
ENV TZ=Asia/Shanghai 
  
#将walle-cli等jar包拷贝到容器内部
ADD ./lib/ /sdk 
RUN chmod -R 775 /sdk

#处理腾讯cos
RUN mkdir /sdk/cos \
     && cd /sdk/cos \
     && curl -o coscli-linux $COS_CLI_URL \
     && mv coscli-linux coscli \
     && chmod 755 coscli

# .cos.yaml 拷贝到 /sdk/cos
ADD ./.cos.yaml /sdk/cos/
#检查cos
RUN /sdk/cos/coscli ls cos://app-pkg-1254950508 -c /sdk/cos/.cos.yaml

# 设置变量 JAVA_HOME 已经在path中
ENV JAVA_HOME="/opt/java/openjdk"
ENV ANDROID_HOME="/sdk/android/sdk" \
     SDK_TOOL_URL="https://dl.google.com/android/repository/commandlinetools-linux-8512546_latest.zip" \
     COS_CLI_URL="https://cosbrowser-1253960454.cos.ap-shanghai.myqcloud.com/software/coscli/coscli-linux" \
     FLUTTER_HOME="/sdk/flutter"\
     FLUTTER_RELEASE_URL="https://storage.flutter-io.cn/flutter_infra_release/releases/stable/linux/flutter_linux_3.3.10-stable.tar.xz"

# 创建android sdk目录,并下载 sdkmanager
RUN mkdir -p ${ANDROID_HOME} \
     && mkdir -p $FLUTTER_HOME \
     && cd $ANDROID_HOME \
     && curl -o sdk.zip $SDK_TOOL_URL \
     && unzip sdk.zip \
     && rm sdk.zip

# 安装android sdk其他package, 输入y是因为此处会有一个licence,需要用户同意后才会安装 
RUN echo yes | ${ANDROID_HOME}/cmdline-tools/bin/sdkmanager --sdk_root=${ANDROID_HOME} --licenses
RUN echo y | ${ANDROID_HOME}/cmdline-tools/bin/sdkmanager --sdk_root=${ANDROID_HOME} "platform-tools" "platforms;android-29" "build-tools;29.0.3"
RUN echo y | ${ANDROID_HOME}/cmdline-tools/bin/sdkmanager --sdk_root=${ANDROID_HOME} "platforms;android-28" "build-tools;28.0.3" "build-tools;28.0.2"
RUN echo y | ${ANDROID_HOME}/cmdline-tools/bin/sdkmanager --sdk_root=${ANDROID_HOME} "platforms;android-30" "build-tools;30.0.0" "build-tools;30.0.2" "build-tools;30.0.3"
RUN echo y | ${ANDROID_HOME}/cmdline-tools/bin/sdkmanager --sdk_root=${ANDROID_HOME} "platforms;android-31" "build-tools;31.0.0"

#更换一个软件源
RUN sed -i -e "s/deb.debian.org/mirrors.tuna.tsinghua.edu.cn/" /etc/apt/sources.list
#安装flutter需要的插件
RUN apt-get update && \
    apt-get install -y \
       xz-utils
    
# 设置环境变量: 把 android sdk 路径加入到 PATH 中
ENV PATH="$PATH:$ANDROID_HOME:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools:$FLUTTER_HOME/bin:$FLUTTER_HOME/bin/cache/dart-sdk/bin"

#flutter
RUN echo "Flutter sdk" && \
    cd /sdk && \
    curl -o flutter.tar.xz $FLUTTER_RELEASE_URL && \
    tar xf flutter.tar.xz && \
    git config --global --add safe.directory $FLUTTER_HOME && \
    flutter config --no-analytics && \
    rm -f flutter.tar.xz   
    
# Flutter 设定镜像
ENV PUB_HOSTED_URL=https://pub.flutter-io.cn
ENV FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn

#创建gradle_cache目录，在job内由gradlew指定项目的gradle缓存目录，否则cleanworkspace之后，缓存不在了,/var/jenkins_home被映射到host了，一直存在。

#gradle user home
ENV GRADLE_USER_HOME="/var/jenkins_home/gradle_cache"
# 创建jenkins需要的目录。
RUN mkdir -p /var/jenkins_home && \
    chmod 777 /var/jenkins_home 	&& \
    mkdir /var/jenkins_home/gradle_cache 
    
    
#将gradle全局属性拷贝到GRADLE_USER_HOME ，方便全局设置gradle相关的配置，gradle的jvm参数，以及全局的maven地址
ADD ./init.gradle $GRADLE_USER_HOME
ADD ./gradle.properties $GRADLE_USER_HOME

#使用jenkins内嵌的cli 安装插件 
RUN jenkins-plugin-cli --plugins build-name-setter:2.2.0 git-parameter:0.9.17 envinject:2.892.v25453b_80e595 thinBackup:1.15 pipeline-utility-steps:2.13.0 versionnumber:1.10 build-user-vars-plugin:1.9

USER jenkins
```

**需要注意的几个点：**，

- `jenkins-plugin-cli` 需要连接jenkins plugin.io，可能会下载plugin失败，jenkins启动之后需要检查一下对应的插件是否安装。
-  JAVA_HOME= /opt/java/openjdk 
- ANDROID_HOME=/sdk/android/sdk
- FLUTTER_HOME=/sdk/flutter
- GRADLE_USER_HOME=/var/jenkins_home/gradle_cache



# 2.配置docker-compose.yml

```shell
version: "3.4"
services:
  jenkins:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: jenkins
    restart: always
    environment:
      JAVA_OPTS: "-server -Djava.awt.headless=true -Duser.timezone=Asia/Shanghai"
    ports:
      - "80:8080"
      - "50000:50000"
    volumes:
      - "jenkins-data:/var/jenkins_home"
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
```

这里使用了一个docker volumes "jenkins-data".

可以提前创建好

```shell
docker volume create --name=jenkins-data
```

# 3.启动容器

```sh
docker compose up -d
```

