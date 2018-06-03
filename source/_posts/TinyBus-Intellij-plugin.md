---
title: TinyBus Intellij plugin
date: 2017-10-14 12:12:56
tags: TinyBus IntellijPlugin eventbus
---

由于项目里使用到了TinyBus事件总线库，想要post方法到subscribe方法的互相导航，在网上搜索了一圈之后，找不到类似eventbus-interllij-plugin的插件，自己简单的更改了 eventbus-interllij-plugin的代码，实现了tinybus-interllij-plugin。该插件并没有修改plugin id，所以不能和eventbus-interllij-plugin同时安装，感谢eventbus-interllij-plugin作者的开源。

先上图

![tinybus-intellij-plugin](http://oav23hfp9.bkt.clouddn.com/17-10-13/40809369.jpg)


目前只支持如下几种导航

- <code>TinyBus.post</code> to <code>@Subscribe</code> or <code>onXxxEvent</code>

- <code>TinyBus.postDelay</code> to <code>@Subscribe</code> or <code>onXxxEvent</code>

- <code>@Subscribe</code> to <code>TinyBus.post</code>

- 不支持<code>@Subscribe</code> to <code>TinyBus.postDelay</code>


<a href="http://oav23hfp9.bkt.clouddn.com/tinybus-intellij-plugin.jar">插件下载地址</a>