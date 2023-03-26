---
title: jenkins pipeline实战——企业级Android自动化打包
categories:
  - - android
  - - android线上填坑
  - - android studio
  - - flutter
  - - compose
tags:
  - - android
date: 2023-02-18 11:15:44
---



# 参考文档 

​	1. [Moving from buddybuild to Jenkins for Android Developers](https://www.jenkins.io/zh/blog/2018/01/08/moving-from-buddybuild-for-android/) 官网jenkins for android

# 简介

在上一篇文章中已经配置好了jenkins [docker搭建Jenkins+Android构建环境 | yndongyong‘s blog](https://yndongyong.github.io/2023/02/15/docker搭建Jenkins-Android构建环境/)，这里主要记录如何通过jenkins pipeline 构建一个满足自己需求的打包job。

目前公司的App，一次完整的打包路径为：

**拉取代码** -> **修改版本名/版本号/flutter aar版本** -> **打包** -> **加固** -> **32渠道包** -> **64渠道包**->**64位测试包上传cos/内测平台**->**企业微信通知**->**32包上传cos** 

产品App单次全量构建的打包时间基本在半个小时左右（主要还跑着其他服务），主要是耗时在打包和加固 这两个阶段。其中加固服务使用的第三方服务耗时不可控。

这一套ci流程对于公司的所有App基本固定，但还是经历了3个阶段，才慢慢演变到当前的脚本。

ci流程，最初的阶段是使用jenkins的自由风格，这一个阶段所有的app的打包流程配置各不相同是主要的问题。第二阶段转换为了pipeline，但是不支持从整个链路中间的的某一步骤重启的，譬如加固失败了，想要从这一步重新开始在这个阶段是不支持的，只能重头开始执行。第三个阶段，统一了公司所有app的分支管理规范，将所有App的打包流程都固化到了这一套流程上来了，支持从指定步骤重启，方便那些失败的步骤，节省构建时间，支持gradle构建缓存，增量编译。

# 特性

目前这一套ci流程，主要支持以下特性：

- 支持指定仓库地址、指定分支
- 修改versionCode versionName、修改flutter aar版本号（可选）
- 打包：支持指定打包类型，alpha:测试;release:正式包
- 加固：可选，目前只有产品App需要加固，其他项目类型的app不需要加固，使用的三方加固，将32位包和64位包上传到第三方平台加固，等待加固完成之后再下载回来。
- 32为渠道包，其中还包括，重新签名，使用walle生成各个应用商店的渠道包。
- 64为渠道包，先重新签名，再使用walle生成各个应用商店的渠道包。
- 64位测试包上传cos/内测平台：用于测试的包上传cos，提交到内测平台
- 企业微信通知：新包构建成功到企业微信群里。
- 32位包上传cos：将应用商店的渠道包上传cos指定目录，用于应用内更新下载地址。
- 支持从任一步骤重启整个流程。
- gradle构建缓存从job的workspace分离出去了，所以支持清空workspace之后也能用到gradle build cache

# 脚本

脚本编写

1. 抽取各个项目的不同点
2. 配置参数话构建
3. 使用声明式的pipeline
4. 涉及的jenkinsplugin参考[docker搭建Jenkins+Android构建环境 | yndongyong‘s blog](https://yndongyong.github.io/2023/02/15/docker搭建Jenkins-Android构建环境/)****

jenkins pipeline脚本编写

算了，详细的还是看脚本吧，其中遗留着一些googleplay商店渠道包、上传fir的脚本。

测试包的文件名：`AKP_FILE_PREFIXX_${"yyyy-MM-dd"}.${BUILDS_TODAY}_channel.apk`

正式包的文件名：`AKP_FILE_PREFIXX_VERSION_NAME.500_channel.apk`

构建包时指定了gradle的构建缓存目录为jenkinshome/jobname文件夹路径

```shell
./gradlew :app:assembleNormalRelease --project-cache-dir $__GRADLE_CACHE_PATH/$JOB_NAME	
```

每个App打包job需要改动的地方都放在了开头部分，其余都为通用逻辑.

```groovy
//NOTICE:配置各个App需要修改的部分 start ====================================
//apk名称前缀
String __AKP_FILE_PREFIXX = "appname_"
//代码库
String __GIT_REPO = 'https://xxxxx/xxxxxx.git'
//jenkins git 凭据id
String __GIT_CREDENTIALS_ID = "credentials_id_name"
//默认versionName
String __VERSION_NAME = "1.2.1"
//默认versionCode
String __VERSION_CODE = "10"
//cos bucket
String __COS_BUCKET_NAME = 'app-bucket-xxxx'
//梆梆加固策略策略id
__BANGBANG_ID = xxxx   
//内测平台的apitoken projectId
String __FIR_API_KEY_RELEASE = "xxxx"  //正式apk使用的projectid
String __FIR_API_KEY_DEBUG = "xxxxx"  // 测试包使用的projectid
// 内测平台 的地址
__FIR_URL_ENPOINT = 'https://xxxxx/create_android'

//渠道号
String CHANNEL_LIST = 'official,yingyongbao,huawei,xiaomi,lianxiang,vivo,oppo'
//企业微信的机器人
String __WX_ROBOT_RELEASE_URL = "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxxxxxx" //正式
String __WX_ROBOT_DEBUG_URL   = "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxxxxxx" //测试

//NOTICE:需要修改的部分 end ====================================
```

整的pipeline脚本如下：

```groovy
/* groovylint-disable LineLength */

//NOTICE:配置各个App需要修改的部分 start ====================================
//apk名称前缀
String __AKP_FILE_PREFIXX = "appname_"
//代码库
String __GIT_REPO = 'https://xxxxx/xxxxxx.git'
//jenkins git 凭据id
String __GIT_CREDENTIALS_ID = "credentials_id_name"
//默认versionName
String __VERSION_NAME = "1.2.1"
//默认versionCode
String __VERSION_CODE = "10"
//cos bucket
String __COS_BUCKET_NAME = 'app-bucket-xxxx'
//梆梆加固策略策略id
__BANGBANG_ID = xxxx   
//内测平台的apitoken projectId
String __FIR_API_KEY_RELEASE = "xxxx"  //正式apk使用的projectid
String __FIR_API_KEY_DEBUG = "xxxxx"  // 测试包使用的projectid
// 内测平台 的地址
__FIR_URL_ENPOINT = 'https://xxxxx/create_android'

//渠道号
String CHANNEL_LIST = 'official,yingyongbao,huawei,xiaomi,lianxiang,vivo,oppo'
//企业微信的机器人
String __WX_ROBOT_RELEASE_URL = "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxxxxxx" //正式
String __WX_ROBOT_DEBUG_URL   = "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxxxxxx" //测试

//NOTICE:需要修改的部分 end ====================================

/*BUILD_TYPE_SELECT: alpha:用于测试，开启了加密各种,release:正式包*/
String BUILD_TYPE_RELEASE ="Release"
String BUILD_TYPE_ALPHA ="alpha"
String BUILD_TYPE_DEBUG ="debug"

//项目的本地缓存路径，将gradle构建目录分离出workspace，否则一单清理worksapce，就需要全量构建
String __GRADLE_CACHE_PATH = "/var/jenkins_home/.gradle"

//调试任务使用，不跳过stage ; true：不跳过 false :跳过 
Boolean __DONT_SKIP_STAGE = true

pipeline {
    agent any
    parameters {
        // 渠道市场
        string(name: 'VERSION_NAME', defaultValue: "$__VERSION_NAME", description: 'build.gradle 中 versionName 字段')
        string(name: 'VERSION_CODE', defaultValue: "$__VERSION_CODE", description: 'build.gradle 中 versionCode 字段')
        string(name: 'CHANNELS', defaultValue: "$CHANNEL_LIST", description: '应用商店渠道号')
        choice(name: 'BUILD_TYPE_SELECT', choices: ["$BUILD_TYPE_RELEASE","$BUILD_TYPE_ALPHA"], description: '构建类型:\nalpha:用于测试，开启了加密、混淆、sentry各种\nrelease:正式包')
        gitParameter( branchFilter: 'origin/release/.*', defaultValue: "origin/release/v$__VERSION_NAME", name: 'BUILD_GIT_BRANCH', type: 'PT_BRANCH', selectedValue: 'TOP', sortMode: 'DESCENDING_SMART', useRepository: "$__GIT_REPO")
        booleanParam(name: 'IS_CLEAN_BUILD', defaultValue: false, description: '是否需要全量构建（包含clean && build）,正式包默认会clean')
        booleanParam(name: 'IS_CLEAN_WORKSPACE', defaultValue: false, description: '是否需要清空WORKSPACE')
        booleanParam(name: 'IS_NEED_JIAGU', defaultValue: true, description: '包是否需要加固')
        string(name: 'FLUTTER_AAR_VERSION_NAME', defaultValue: "", description: 'flutter aar产物的版本号')
    }
    environment {
        buildToolsPath = "$ANDROID_HOME/build-tools/29.0.3"
        //NOTICE:
        //walleLibPath = '/walle' ci环境  本地打包环境：'/sdk/walle'
        walleLibPath = '/sdk/walle'
        cosLibPath = '/sdk/cos'
        PATH = "$PATH:$buildToolsPath"

        //设置各个build_type下需要用的参数
        __API_KEY = getShEchoResult("""
            if [ $params.BUILD_TYPE_SELECT = $BUILD_TYPE_RELEASE ];then
                echo "$__FIR_API_KEY_RELEASE"
            else
                echo "$__FIR_API_KEY_DEBUG"            
            fi
            """)
        //debug
        __BUILD_TYPE = getShEchoResult("""
            if [ $params.BUILD_TYPE_SELECT = $BUILD_TYPE_RELEASE ];then
                echo "release"
            elif [ $params.BUILD_TYPE_SELECT = $BUILD_TYPE_ALPHA ];then
                echo "alpha"    
            else
                echo "debug"            
            fi
            """)
        
        //wechate notify url 企业微信打包机器人
        __WEIXIN_SEND_URL = getShEchoResult("""
            if [ $params.BUILD_TYPE_SELECT = $BUILD_TYPE_RELEASE ];then
                echo "$__WX_ROBOT_RELEASE_URL"
            else
                echo "$__WX_ROBOT_DEBUG_URL"            
            fi
        """)

        

        
        //版本号
        __versionNumberOfToday = VersionNumber(versionNumberString: '${BUILD_DATE_FORMATTED,"yyyy-MM-dd"}.${BUILDS_TODAY}',worstResultForIncrement: 'SUCCESS');
        NumberVersion = getShEchoResult("""
            if [ $params.BUILD_TYPE_SELECT = $BUILD_TYPE_RELEASE ];then
                echo "${params.VERSION_NAME}.500"
            else
                echo "${params.VERSION_NAME}.${__versionNumberOfToday}"            
            fi
        """)

        //重写这些值，得用with函数
        __CHANGE_LOG = ""
       
        // 定义项目编译完成之后的包文件文件名
        // arm32Apk="""${sh(returnStdout: true, script: "echo $WORKSPACE/${project}/bin/arm32/${__AKP_FILE_PREFIXX}-*.apk")}"""
    }

    options {
        timestamps() // 在日志中打印时间
    }
    stages {
        stage('克隆代码') {
            when{
                expression {
                    return __DONT_SKIP_STAGE
                }
            }
            steps {
                // wrap([$class: 'BuildUser']) {
                //     script {
                //         BUILD_USER = "${env.BUILD_USER}"
                //     }
                // }
                //20-release/v1.0.0-alpha-user
                buildName "#${BUILD_NUMBER}-${params.BUILD_GIT_BRANCH}-${params.BUILD_TYPE_SELECT}-${env.BUILD_USER}"
                
                //克隆
                script{
                    if (params.IS_CLEAN_WORKSPACE == true) {
                            cleanWs()
                    }
                    def scmVars  = checkout([$class: 'GitSCM', branches: [[name: "${params.BUILD_GIT_BRANCH}"]], extensions: [], userRemoteConfigs: [[credentialsId: "${__GIT_CREDENTIALS_ID}", url: "${__GIT_REPO}"]]])
                    env.GIT_COMMIT = scmVars.GIT_COMMIT
                    env.GIT_PREVIOUS_SUCCESSFUL_COMMIT = scmVars.GIT_PREVIOUS_SUCCESSFUL_COMMIT
                }
               
            }
        }
        stage('修改版本名/版本号/flutter aar版本') {
            when{
                expression {
                    return __DONT_SKIP_STAGE
                }
            }
            steps {
                script {
                    echo '环境/参数准备'
                    sh("""
                        if [ ! -d "./bin" ];then
                            mkdir bin
                        fi
                        if [ ! -d "./bin/temp" ];then
                            mkdir bin/temp
                        fi
                        if [ ! -d "./bin/temp32" ];then
                            mkdir bin/temp32
                        fi
                        if [ ! -d "./bin/temp64" ];then
                            mkdir bin/temp64
                        fi

                        if [ ! -d "./bin/arm32" ];then
                            mkdir bin/arm32
                        fi
                        if [ ! -d "./bin/arm64" ];then
                            mkdir bin/arm64
                        fi
                        rm ./bin/*.zip
                    """) 
                    
                    
                    def oldVerserionName =getShEchoResult( """
                                grep "versionName" "app/build.gradle" | sed -e 's/^[ \t]*//;s/[ \t]*\$//' | cut -f2 -d ' ' | sed -e 's/\"//g'
                                """)
                    echo "oldVerserionName:$oldVerserionName"
                    sh ("""
                        sed -i "app/build.gradle" -e 's/versionName "${oldVerserionName}"/versionName "${NumberVersion}"/g'
                        """)

                    def oldVersionCode =  getShEchoResult("""
                                grep "versionCode" "app/build.gradle" | sed -e 's/^[ \t]*//;s/[ \t]*\$//' | cut -f2 -d ' ' | sed -e 's/\"//g'
                                """)
                    echo "oldVersionCode:$oldVersionCode"
                    sh (""" sed -i "app/build.gradle" -e "s/versionCode ${oldVersionCode}/versionCode ${params.VERSION_CODE}/g" """)

                    newVersionCode =  getShEchoResult("""
                                grep "versionCode" "app/build.gradle" | sed -e 's/^[ \t]*//;s/[ \t]*\$//' | cut -f2 -d ' ' | sed -e 's/\"//g'
                                """)

                    newVerserionName =getShEchoResult( """
                                grep "versionName" "app/build.gradle" | sed -e 's/^[ \t]*//;s/[ \t]*\$//' | cut -f2 -d ' ' | sed -e 's/\"//g'
                                """)

                    echo "new versionName:$newVersionCode"
                    echo "new versionCode:$newVerserionName"
                    if (newVerserionName != NumberVersion) {
                        error '修改版本名称失败'
                    }
                    if (newVersionCode != params.VERSION_CODE) {
                        error '修改版本号失败'
                    }
                    //修改 flutter aar的版本号
                    if(params.FLUTTER_AAR_VERSION_NAME){
                        def oldFlutterAarVerserionName =getShEchoResult( """
                                grep "def flutter_aar_sdk_version" "app/build.gradle" | sed -e 's/^[ \t]*//;s/[ \t]*\$//' | cut -f2 -d '=' | sed -e 's/\"//g'
                                """)
                        echo "oldFlutterAarVerserionName:$oldFlutterAarVerserionName"
                        if(oldFlutterAarVerserionName){
                            if(params.FLUTTER_AAR_VERSION_NAME != oldFlutterAarVerserionName){
                                sh ("""
                                sed -i "app/build.gradle" -e 's/def flutter_aar_sdk_version = "${oldFlutterAarVerserionName}"/def flutter_aar_sdk_version = "${params.FLUTTER_AAR_VERSION_NAME}"/g'
                                """)
                                def newlutterAarVerserionName =getShEchoResult( """
                                    grep "def flutter_aar_sdk_version" "app/build.gradle" | sed -e 's/^[ \t]*//;s/[ \t]*\$//' | cut -f2 -d '=' | sed -e 's/\"//g'
                                    """)
                                echo "new flutter_aar_sdk_version:$newlutterAarVerserionName" 
                                if (params.FLUTTER_AAR_VERSION_NAME != newlutterAarVerserionName) {
                                    error '修改flutter aar sdk version失败'
                                }
                                echo "修改flutter aar sdk version成功，新版本：${newlutterAarVerserionName}"
                                //提交改动的文件
                                
                                // GIT_CREDS = credentials(${__GIT_CREDENTIALS_ID})
                                // sh("""
                                   
                                //     git add app/build.gradle
                                //     git commit -m "feat(flutter):升级版本flutter aar版本号为：${params.FLUTTER_AAR_VERSION_NAME}"
                                //     git push
                                // """)
                                def fixFlutterAarLog = "修改flutter aar version：${newlutterAarVerserionName}"
                                withEnv(["__CHANGE_LOG=$fixFlutterAarLog"]) { 
                                    echo "${env.__CHANGE_LOG}" 
                                }
                            }else{
                                echo "app中的flutter aar sdk version已近是最新无需修改，版本：${oldFlutterAarVerserionName}"
                            }           
                        }        
                        
                    }
                    // 修改release的v2签名先默认关闭，等加固之后再使用v2签名重新签名
                    if(params.IS_NEED_JIAGU){
                        def signStr =getShEchoResult( """
                                grep "v2SigningEnabled true" "app/build.gradle" | sed -e 's/^[ \t]*//;s/[ \t]*\$//' | sed -n 1p
                                """)
                        echo "app build sign:$signStr"
                        if(signStr == "v2SigningEnabled true"){
                            sh(""" sed -i 'app/build.gradle' -e 's/v2SigningEnabled true/v2SigningEnabled false/g'  """)
                            def signStrNew =getShEchoResult( """
                                grep "v2SigningEnabled false" "app/build.gradle" | sed -e 's/^[ \t]*//;s/[ \t]*\$//' | sed -n 1p
                                """)
                            if(signStrNew=="v2SigningEnabled false"){
                                echo "修改签名:$signStrNew" 
                            }   
                        }else{
                            echo "不需要修改签名方式:$signStr"
                        } 
                        
                                    
                    }
                }
            }
        }
        stage('打包') {
            when{
                expression {
                    return __DONT_SKIP_STAGE
                }
            }
            steps {
                script {
                    sh 'chmod +x gradlew'
                    if (params.IS_CLEAN_BUILD == true) {
                        sh './gradlew clean'
                    }
                    echo "Build ${params.BUILD_TYPE_SELECT} APK.. "
                    //指定了gradle的构建缓存目录 --project-cache-dir $__GRADLE_CACHE_PATH/$JOB_NAME
                    if(params.BUILD_TYPE_SELECT == "$BUILD_TYPE_ALPHA"){
                        sh "./gradlew :app:aNA -PIS_LOCAL_BUILD=false --project-cache-dir $__GRADLE_CACHE_PATH/$JOB_NAME"
                    }else if(params.BUILD_TYPE_SELECT == "$BUILD_TYPE_DEBUG"){
                        sh "./gradlew :app:aND --project-cache-dir $__GRADLE_CACHE_PATH/$JOB_NAME"
                    }else{
                        sh "./gradlew :app:assembleNormalRelease --project-cache-dir $__GRADLE_CACHE_PATH/$JOB_NAME"
                    }
                }
            }
        }
        stage('加固') {
            when{
                expression {
                    return __DONT_SKIP_STAGE && params.IS_NEED_JIAGU && params.BUILD_TYPE_SELECT != "$BUILD_TYPE_DEBUG" 
                }
            }
            steps {
                script {
                    def currentPath = pwd()
                    
                    def protectedSourceApkName32 = "${currentPath}/bin/${__AKP_FILE_PREFIXX}32_${NumberVersion}_release.apk"
                    def protectedSourceApkName64 = "${currentPath}/bin/${__AKP_FILE_PREFIXX}64_${NumberVersion}_release.apk"
                    
                    sh ('''
                        rm -rf ./bin/*.apk
                        rm -rf ./bin/temp/*
                        ''')
                    //copy 32 to bin
                    sh (""" cp app/build/outputs/apk/normal/${__BUILD_TYPE}/*-v7a-${__BUILD_TYPE}.apk "${protectedSourceApkName32}" """)
                    legu32Apk = "${currentPath}/bin/temp/${__AKP_FILE_PREFIXX}32_${NumberVersion}_release_sec.apk"

                    //copy 64 to bin
                    sh (""" cp app/build/outputs/apk/normal/${__BUILD_TYPE}/*-v8a-${__BUILD_TYPE}.apk "${protectedSourceApkName64}" """)
                    legu64Apk = "${currentPath}/bin/temp/${__AKP_FILE_PREFIXX}64_${NumberVersion}_release_sec.apk"

                    //加固
                    executeJiagu("${currentPath}/bin", legu32Apk,legu64Apk, "${currentPath}/bin/temp")

                }
            }
        }
        stage('32渠道包') {
            when{
                expression {
                    return __DONT_SKIP_STAGE && params.IS_NEED_JIAGU && params.BUILD_TYPE_SELECT != "$BUILD_TYPE_DEBUG"
                }
            }
            steps {
                script {
                    def currentPath = pwd()
                    def zipalignedApkPath32 = "./bin/temp32/${__AKP_FILE_PREFIXX}${NumberVersion}_aligned.apk"
                    def signedApkPath32 = "./bin/temp32/${__AKP_FILE_PREFIXX}${NumberVersion}_android.apk"
                    def archived32Path = "${currentPath}/bin/arm32"
                    legu32Apk = "${currentPath}/bin/temp/${__AKP_FILE_PREFIXX}32_${NumberVersion}_release_sec.apk"

                    sh ('''
                            rm -rf ./bin/temp32/*
                            rm -rf ./bin/arm32/*
                            ''')
                    //渠道包签名
                    executeWalleSigner(signedApkPath32, legu32Apk, zipalignedApkPath32, archived32Path)
                    //归档渠道包
                    sh ("""
                            zip -j arm32_channels.zip ${archived32Path}/${__AKP_FILE_PREFIXX}${NumberVersion}_*.apk
                            mv arm32_channels.zip bin/
                          """)  
               }
                archiveArtifacts artifacts: 'bin/arm32_channels.zip', followSymlinks: false, onlyIfSuccessful: true
                archiveArtifacts artifacts: "bin/arm32/*_${NumberVersion}*_android_official.apk", followSymlinks: false, onlyIfSuccessful: true
            }
            
        }
        stage('64渠道包') {
            when{
                expression {
                    return __DONT_SKIP_STAGE && params.IS_NEED_JIAGU && params.BUILD_TYPE_SELECT != "$BUILD_TYPE_DEBUG"
                }
            }
            steps {
                script {
                    def currentPath = pwd()
                    def zipalignedApkPath64 = "./bin/temp64/${__AKP_FILE_PREFIXX}${NumberVersion}_aligned.apk"
                    signedApkPath64 = "./bin/temp64/${__AKP_FILE_PREFIXX}${NumberVersion}_android.apk"
                    def archived64Path = "${currentPath}/bin/arm64"

                    legu64Apk = "${currentPath}/bin/temp/${__AKP_FILE_PREFIXX}64_${NumberVersion}_release_sec.apk"
                    sh (""" rm -rf ./bin/temp64/*
                            rm -rf ./bin/arm64/*
                       """)
                    //渠道包签名
                    executeWalleSigner(signedApkPath64, legu64Apk, zipalignedApkPath64, archived64Path)
                    //归档渠道包
                    sh ("""
                        zip -j arm64_channels.zip ${archived64Path}/${__AKP_FILE_PREFIXX}${NumberVersion}_*.apk
                       mv arm64_channels.zip bin/
                      """)

                }
                archiveArtifacts artifacts: 'bin/arm64_channels.zip', followSymlinks: false, onlyIfSuccessful: true

            }
            
        }
        stage('64位测试包上传cos/分发平台') {
            when{
                expression {
                    return __DONT_SKIP_STAGE
                }
            }
            steps {
                script {
                    echo '上传cos'
                    def currentPath = pwd()
                    // //上传到cos，企业微信通知到下载连接 debug 包
                    if(params.BUILD_TYPE_SELECT == "$BUILD_TYPE_DEBUG" || params.IS_NEED_JIAGU == false){
                        def apkName = "${__AKP_FILE_PREFIXX}${NumberVersion}_${__BUILD_TYPE}.apk"
                        def tempApkPath = "${currentPath}/bin/arm64/${apkName}"
                        sh (""" cp app/build/outputs/apk/normal/${__BUILD_TYPE}/*-v8a-${__BUILD_TYPE}.apk "${tempApkPath}" """)
                        sh(""" ${cosLibPath}/coscli cp ${tempApkPath} cos://${__COS_BUCKET_NAME}/test/${apkName} -c ${cosLibPath}/.cos.yaml """)
                        __APK_DOWNLOAD_URL = "https://app-pkg-1254950508.cos.ap-guangzhou.myqcloud.com/test/${apkName}"
                    }else{
                        //有加固的包
                        //从arm64的包下面找
                         def tempApkPath = "bin/arm64/${__AKP_FILE_PREFIXX}*_official.apk"
                         def apkFiles = findFiles(glob: "${tempApkPath}")
                         if(apkFiles[0].name == ""){
                            error "上传cos 的apk 没有找到"
                        }
                         sh(""" ${cosLibPath}/coscli cp ${apkFiles[0].path} cos://${__COS_BUCKET_NAME}/test/${apkFiles[0].name} -c ${cosLibPath}/.cos.yaml """)
                        __APK_DOWNLOAD_URL = "https://app-pkg-1254950508.cos.ap-guangzhou.myqcloud.com/test/${apkFiles[0].name}"
                        
                    }
                    try{
                        retry(3){
                            if(params.BUILD_TYPE_SELECT == "$BUILD_TYPE_RELEASE"){
                                uploadApkToInternal("正式版")
                            }else{
                                uploadApkToInternal("内测版")
                            } 
                        }
                    }catch(Exception e){
                        print e
                    }
                    buildDescription "构建成功下载：<a href=\"${__APK_DOWNLOAD_URL}\">下载</a>"
                   
                }
            }
        }
        stage('企业微信通知'){
            when{
                expression {
                    return __DONT_SKIP_STAGE
                }
            }
            steps {
                script {
                    echo '企业微信通知'
                    // sh "printenv"
                    //企业微信通知
                    def gitLog = getGitChangeLogBetweenLastBuildSucceesCommit()
                    if(!gitLog){
                        gitLog = getChangeString()
                    }
                    if(!gitLog){
                        //__CHANGE_LOG 是定义在evn中，不能用脚本式的方式修改
                        withEnv(["__CHANGE_LOG=$gitLog\n${env.__CHANGE_LOG}"]) { 
                            echo "${env.__CHANGE_LOG}" 
                        }
                    }
                    qyWechatNotify()
                }
            }
        }
        stage('32上传cos') {
            when{
                expression {
                    return __DONT_SKIP_STAGE && params.BUILD_TYPE_SELECT == "$BUILD_TYPE_RELEASE" && params.IS_NEED_JIAGU
                }
            }
            steps {
                script {
                    def currentPath = pwd()
                    //cos这里的文件夹名对应版本号，配置后台的格式 6.2.0.500
                    sh(""" ${cosLibPath}/coscli cp ${currentPath}/bin/arm32/ cos://${__COS_BUCKET_NAME}/${NumberVersion}/ -r -c ${cosLibPath}/.cos.yaml """)
                }
                
            }
        }
        // stage('google渠道包') {
        //     when{
        //         expression {
        //             return params.BUILD_TYPE_SELECT == "$BUILD_TYPE_RELEASE"
        //         }
        //     }
        //     steps {
        //         echo 'google渠道包'
        //             script {
        //                 def currentPath = pwd()
                        
        //                 archived32Path = "${currentPath}/bin/arm32"
        //                 legu32Apk = "${currentPath}/bin/temp/${__AKP_FILE_PREFIXX}32_${NumberVersion}_release_sec.apk"
        //                 def protectedSourceApkName32 = "${currentPath}/bin/${__AKP_FILE_PREFIXX}32_${NumberVersion}_release.apk"
        //                 def protectedSourceApkName64 = "${currentPath}/bin/${__AKP_FILE_PREFIXX}64_${NumberVersion}_release.apk"
                        
        //                 sh ('''
        //                     rm -rf ./bin/*.apk
        //                     ''')
        //                 //copy 32 to bin
        //                 sh (""" cp app/build/outputs/apk/normal/${__BUILD_TYPE}/*-v7a-${__BUILD_TYPE}.apk "${protectedSourceApkName32}" """)

        //                 sh (""" cp app/build/outputs/apk/normal/${__BUILD_TYPE}/*-v8a-${__BUILD_TYPE}.apk "${protectedSourceApkName64}" """)

        //                 def googleApkName32 = "${currentPath}/bin/google/${__AKP_FILE_PREFIXX}32_${NumberVersion}_release.apk"
        //                 def googleApkName64 = "${currentPath}/bin/google/${__AKP_FILE_PREFIXX}64_${NumberVersion}_release.apk"
        //                 //签名
        //                 executeWalleSignerForGoogle(protectedSourceApkName32,googleApkName32)
        //                 executeWalleSignerForGoogle(protectedSourceApkName64,googleApkName64)
        //                  //归档渠道包
        //                 sh ("""
        //                     zip -j google_channels.zip ${pwd()}/bin/google/${__AKP_FILE_PREFIXX}*.apk
        //                     mv google_channels.zip bin/
        //                 """)

        //             }
        //             archiveArtifacts artifacts: "bin/google_channels.zip", followSymlinks: false, onlyIfSuccessful: true
        //     }
        // }
        
        
    }

    post{
        always {
            //为重启执行的步骤设置buildName
            buildName "#${BUILD_NUMBER}-${params.BUILD_GIT_BRANCH}-${params.BUILD_TYPE_SELECT}-${env.BUILD_USER}"
        }   
        
    }
}
// 获取 shell 命令输出内容
def getShEchoResult(cmd) {
    def getShEchoResultCmd = "ECHO_RESULT=`${cmd}`\necho \${ECHO_RESULT}"
    return sh (
        script: getShEchoResultCmd,
        returnStdout: true
    ).trim()
}
//加固
// protectedSourceApkName 带加固的文件夹路径
// jiaguApk 加固之后的文件
// downloadPath 梆梆加固之后下载的路径
def executeJiagu(protectedSourceApkName, jiagu32Apk,jiagu64Apk, downloadPath) {
    echo '加固start'
    // sh( """ ${leguLibPath}/jiagu "${protectedSourceApkName}" "${jiaguApk}" /legu/base.conf """)
    sh(script: """
    java -jar /sdk/bang/secapi.jar -i https://usc.an110.com -u yntyxx -a 62e3ff42-ea46-4139-9b17-764023851612 -c 03b7f84d-ff93-419b-b5be-2c941455fdcd -p ${protectedSourceApkName} -d ${downloadPath} -t ${__BANGBANG_ID} -f 0 """)
    if (fileExists(jiagu32Apk) == false) {
        error '加固失败'
    }
    if (fileExists(jiagu64Apk) == false) {
        error '加固失败'
    }
    echo '加固success'
}

// walle渠道包
def executeWalleSigner(signedApkPath, jiaguApk, zipalignedApkPath, archivedPath) {
    echo '加固之后重新签名' 


    keystorePath = getShEchoResult("""
            grep "storeFile" "app/build.gradle" | sed -e 's/^[ \t]*//;s/[ \t]*\$//' | cut -f2 -d ' ' | sed -e 's/\"//g' | grep -o '[^\\(]/.*[^\\)]' | sed -n 1p
            """)

    keyAlias = getShEchoResult("""
                    grep "keyAlias" "app/build.gradle" | sed -e 's/^[ \t]*//;s/[ \t]*\$//' | cut -f2 -d ' ' | sed -e 's/\"//g' | sed -n 1p
                        """)
    keystorePassword = getShEchoResult("""
                    grep "storePassword" "app/build.gradle" | sed -e 's/^[ \t]*//;s/[ \t]*\$//' | cut -f2 -d ' ' | sed -e 's/\"//g' | sed -n 1p
                        """)
    keyPassword = getShEchoResult("""
                    grep "keyPassword" "app/build.gradle" | sed -e 's/^[ \t]*//;s/[ \t]*\$//' | cut -f2 -d ' ' | sed -e 's/\"//g' | sed -n 1p
                        """)

    sh( """ zipalign -v 4 ${jiaguApk} ${zipalignedApkPath} """)
    //腾讯加固要求摘要 sign --min-sdk-version=17 ,以前的包是v1 v2开启的。
    sh (""" apksigner sign --v1-signing-enabled true --v2-signing-enabled true --ks ${keystorePath} --ks-key-alias ${keyAlias} --ks-pass pass:${keystorePassword} --key-pass pass:${keyPassword} --out ${signedApkPath} ${zipalignedApkPath} """)

    //
    def signApkInfo = getShEchoResult("apksigner verify --verbose ${signedApkPath}")
    signApkInfo = getShEchoResult(""" echo $signApkInfo | sed -n '1,7p' """)
    echo signApkInfo

    echo 'walle渠道包start'
    sh( """ java -jar ${walleLibPath}/walle-cli-all.jar batch -c ${params.CHANNELS} ${signedApkPath} ${archivedPath} """)
    //签名信息检查
    // def apkFiles = findFiles(glob: "${apkFilePath}")
    // def apkPath = apkFiles[0].path
    // sh(returnStdout: true, script: """
    //     apksigner verify -verbose -print-certs \${apkPath} | awk -F \" \" '/Verified using/ {print \${2}}'
    //    """).trim() 
    echo 'walle渠道包success'
}
//google 渠道包
def executeWalleSignerForGoogle(signedApkPath,archivedPath) {
    echo 'google 渠道包start'
    sh( """ java -jar ${walleLibPath}/walle-cli-all.jar put -c google ${signedApkPath} ${archivedPath} """)
    echo 'google 渠道包success'
}
// git log
def getChangeString() {
    MAX_MSG_LEN = 4096
    def changeString = ''

    echo 'Gathering SCM Changes...'
    def changeLogSets = currentBuild.changeSets
    echo "$changeLogSets"
    for (int i = 0; i < changeLogSets.size(); i++) {
        def entries = changeLogSets[i].items
        for (int j = 0; j < entries.length; j++) {
            def entry = entries[j]
            truncated_msg = entry.msg.take(MAX_MSG_LEN)
            changeString += "[${entry.author}] ${truncated_msg}\n"
        }
    }

    if (!changeString) {
        changeString = ''
    }
    return changeString
}

//发送企业微信通知
def qyWechatNotify() {

    def apkFilePath = ""
    apkFilePath = "app/build/outputs/apk/normal/${__BUILD_TYPE}/*-v8a-${__BUILD_TYPE}.apk"
    def apkFiles = findFiles(glob: "${apkFilePath}")
    __APP_NAME = sh(returnStdout: true, script: """
            aapt d badging ${apkFiles[0].path} | grep -E 'application-label:' | sed -e 's/application-label://g' | sed -e \"s/'//g\"
        """).trim()
    __APP_PACKAGE_NAME =sh(returnStdout: true, script: """
            aapt d badging ${apkFiles[0].path} | grep package | awk '{print \$2 }' | sed s/name=//g | sed -e \"s/'//g\"
        """).trim()

    //__APK_DOWNLOAD_URL 下载链接
    def appDownloadUrl = "https://www.akring.online/verify?id=${__API_KEY}"
    def msg = """<font color="warning">[${__APP_NAME}Android更新](${appDownloadUrl})</font>\n>下载链接：[${appDownloadUrl}](${appDownloadUrl})\n>版本: ${NumberVersion}(versionCode: ${params.VERSION_CODE})\n>构建分支: ${params.BUILD_GIT_BRANCH}\n>构建者: ${BUILD_USER}\n>\n>更新内容:${env.__CHANGE_LOG}"""
    
    def result = sh(returnStdout: true, script: """
        curl -H "'Content-type':'application/json'" -d '{"msgtype":"markdown","markdown":{"content":"'"${msg}"'"}}' $__WEIXIN_SEND_URL
    """).trim()
    def qywxResult = readJSON text:result
    if (qywxResult['errcode'] == 0 ) {
        echo '企业微信通知成功'
    }else {
        echo '企业微信通知失败'
    }
}
// 上传apk到fir
def uploadApkToFir(apkFilePath) {
    def apkPath = "app/build/outputs/apk/normal/${__BUILD_TYPE}/*-v8a-${__BUILD_TYPE}.apk"
    def apkS = findFiles(glob: "${apkPath}")
    __APP_NAME = sh(returnStdout: true, script: """
            aapt d badging ${apkS[0].path} | grep -E 'application-label:' | sed -e 's/application-label://g' | sed -e \"s/'//g\"
        """).trim()
    __APP_PACKAGE_NAME =sh(returnStdout: true, script: """
            aapt d badging ${apkS[0].path} | grep package | awk '{print \$2 }' | sed s/name=//g | sed -e \"s/'//g\"
        """).trim()
    //获取token
    def tokenJson = sh(returnStdout: true, script: """
                        curl -H "Content-Type: application/json" -X POST -d '{"api_token": "${__API_KEY}", "bundle_id":"$__APP_PACKAGE_NAME", "type":"android"}' ${__FIR_URL_ENPOINT}
                       """).trim()

    def jsonObj = readJSON text:tokenJson
    echo "$jsonObj"
    def cert = jsonObj['cert']
    if (cert) {
        icon = jsonObj['cert']['icon']

        def key = icon['key']
        def token = icon['token']
        def upload_url = icon['upload_url']
        //upload icon
        def iconPath = "/var/jenkins_home/workspace/$JOB_NAME/app/src/main/res/mipmap-mdpi/ic_launcher.png"

        sh("""
            curl   -F "key=${key}"              \\
                -F "token=${token}"             \\
                -F "file=@${iconPath}"            \\
                ${upload_url}
        """)
        echo 'upload icon success'
        //upload apk
        def apkFiles = findFiles(glob: "${apkFilePath}")

        def apkUPloadReulst = sh(returnStdout: true, script:"""
            curl   -F "key=${jsonObj['cert']['binary']['key']}"              \\
                -F "token=${jsonObj['cert']['binary']['token']}"             \\
                -F "file=@"${apkFiles[0].path}            \\
                -F "x:name=${__APP_NAME}"             \\
                -F "x:version=${NumberVersion}"         \\
                -F "x:build=${params.VERSION_CODE}"               \\
                -F "x:changelog=${__CHANGE_LOG}"       \\
                ${jsonObj['cert']['binary']['upload_url']}
        """).trim()

        def apkResult = readJSON text:apkUPloadReulst
        echo "$apkResult"
        if (apkResult['is_completed'] == true) {
            echo 'apk上传fir成功，开始企业微信通知'
            __APK_DOWNLOAD_URL = "https://${jsonObj['download_domain']}/${jsonObj['short']}"
            buildDescription "构建成功下载：<a href=\"${__APK_DOWNLOAD_URL}\">${__APK_DOWNLOAD_URL}</a>"
        }else {
            echo 'apk上传fir失败'
        }
    }else {
        throw new Exception('上传fir失败：获取token失败')
        currentBuild.result = 'FAILURE'
    }
}
// 提交到内部应用测试分发平台
def uploadApkToInternal(buildTypeName) {
    //获取token
    def appUploadResult = sh(returnStdout: true, script: """
curl -H "Content-Type: application/json" -X POST -d '{"version": "${NumberVersion}", "type":"$buildTypeName","project_id":"$__API_KEY","webhook":"","git_log":"","ipa_url":"$__APK_DOWNLOAD_URL"}' --connect-timeout 60 ${__FIR_URL_ENPOINT}
                       """).trim()

    if("" == appUploadResult){
        echo 'apk上传测试分发平台成功'
    }else{
        echo 'apk上传测试分发平台'
    }

}

//获取直至上一次成功构建之间的commit log ,排除merges信息
def getGitChangeLogBetweenLastBuildSucceesCommit(){
    def appUploadResult = ""
    if("${env.GIT_PREVIOUS_SUCCESSFUL_COMMIT}" != "null"){
        appUploadResult = sh(returnStdout: true, script: """
        git log ${env.GIT_PREVIOUS_SUCCESSFUL_COMMIT}...${env.GIT_COMMIT} --pretty=format:"[%cn] %s"
        """).trim()
    }
    return appUploadResult
}
```

