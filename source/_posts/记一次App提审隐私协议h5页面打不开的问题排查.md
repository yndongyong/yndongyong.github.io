---
title: 记一次App提审隐私协议h5页面打不开的问题排查
categories:
  - - android
  - - android线上填坑
tags:
  - - android
date: 2022-04-29 16:28:00
---

### 问题一

在vivo应用商店上传应用，被拒，提示64、32位的应用隐私协议h5打开之后都是白屏，看拒审通知，两款设备一新一旧，系统版本有Android11、android6。这个h5在本地用个各种浏览器、手机访问都很正常。但是连续多次的提审都因为打开之后白屏被拒。

### 排查

用电脑浏览器、测试手机、手机浏览器打开都很正常，后面只能找前端一起排查问题，h5也是很简单的静态页面，没有用到任何的jsb。之后通过关键字`vivo 打开网页白屏` 网络搜索发现似乎也有很多人遇到，也是反馈vivo系统问题，也没有什么实质性的线索，暂时认为是审核时的网络问题，抱着侥幸的心里，又再一次提审了，结果又一次悲剧了。

之后通过了分析了h5页面，发现引入了几个第三方cnd服务器上的js，基本是vue相关的

```
https://cdn.jsdelivr.net/npm/vue@2.6.14/dist/vue.min.js
https://cdn.jsdelivr.net/npm/vue-router@3.5.2/dist/vue-router.min.js
https://cdn.jsdelivr.net/npm/vuex@3.6.2/dist/vuex.min.js
https://cdn.jsdelivr.net/npm/axios@0.21.1/dist/axios.min.js
https://cdn.jsdelivr.net/npm/vuex@3.6.2/dist/vuex.min.js
```

一开始看很正常，甚至每个连接在云南访问都是很正常，排查webview相关配置异常的问题。但是审核时，网页就是白屏，开始怀疑时审核地那边的网络问题加载不出，vue写的页面，js加载不成功的是不会进行网页的渲染的。再之后通过 `https://tool.chinaz.com/speedtest`站长之家的测速工具发现广东、深圳这一片都无法访问这个域名`cdn.jsdelivr.net` ，同时网上信息表示`jsdelivr`的域名过期了，等我搜索到这一信息的时候，证书已经替换了，但是广东这一片还是访问`jsdeliver`还是超时，破案了。

### 解决办法

最终将上诉的vuejs相关的引入替换为自家服务服务器的js。之后提审正常通过。

### 问题二

好紧不长，之后按要求将App拆为32位、64位的包，之后再在vivo商店上传审核32应用时，总是被拒，提示隐私协议h5打开后展示白屏，发现设备是一款比较老的手机，系统为Android6，webview的内核为58.x。

### 排查

之后在本地使用模拟器（Android6.0的系统）复现了同样的问题，app内的所有网页都是打不开的，由于正式包，能够拦截排查的日志也有限,

```javascript
[chromium] : [INFO:CONSOLE(12)] "The key "viewport-fit" is not recognized and ignored.", source: https://static.ybsjyyn.com/app/about.html#/about/service (12)
 [chromium] : [INFO:CONSOLE(164)] "Uncaught SyntaxError: Unexpected token *", source: https://static.ybsjyyn.com/app/js/chunk-vendors.1e681edf.js (164)
[chromium] : [INFO:CONSOLE(12)] "The key "viewport-fit" is not recognized and ignored.", source: https://static.ybsjyyn.com/app/about.html#/about/service (12)
```

```javascript
[chromium] : [INFO:CONSOLE(1)] "Uncaught TypeError: window._handleMessageFromNative is not a function", source:  (1)
[chromium] : [INFO:CONSOLE(0)] "Unrecognized Content-Security-Policy directive 'worker-src'.
[chromium] : [INFO:CONSOLE(2)] "Uncaught TypeError: Object.values is not a function", source: https://txc.gtimg.com/static/6756/index.e7d6f3d0.js (2)
[chromium] : [INFO:CONSOLE(1)] "Uncaught TypeError: window._handleMessageFromNative is not a function", source:  (1)
[chromium] : [INFO:CONSOLE(1)] "The key "viewport-fit" is not recognized and ignored.", source: https://support.qq.com/embed/phone/30476 (1)
[chromium] : [INFO:CONSOLE(2)] "Uncaught TypeError: Object.values is not a function", source: https://txc.gtimg.com/static/6756/index.e7d6f3d0.js (2)
```

虽然`vue`的文档上说支持Android4.4以上，但是还是怀疑是`vue`的兼容性，一些新的`js`、`css`属性，再低版本系统上缺失。

由于App内的所有网页使用的都是`vue` 让前端团队在这个时候来处理兼容性的已经是不现实了。之后通过分分析`android5，6`的用户只占比0.02%了，将App对android版本支持的下限从`Android5`升级为`Android7`。

再次提审，完美避开了旧版本，正常通过审核。



