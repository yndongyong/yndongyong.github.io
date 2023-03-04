---
title: vscode 使用wepy框架开发小程序的环境配置以及全局配置
categories:
  - - android
  - - android线上填坑
  - - android studio
  - - flutter
  - - compose
tags:
  - - android
date: 2021-08-08 16:29:14
---



1. ### 代码高亮部分按照wepy的提示设置 

2. ### 安装插件 

   1. minapp 
      - template 节点配置lang="wxml" miniapp="wepy" 实现wxml的语法提示和部分wepy的方法的提示,需要输入<之后才能有提示，搭配空格键使用
      - 配置引入全局的样式表 实现css资源的提示 src/components/comm.less（自己的全局的）
   2. Easy LESS 使用了less
   3. Vetur
   4. Wepy
   5. wepy snippets 代码提示，输入wepy
   6. wpy-beautify 实现代码格式化
   7. Vetur-wepy  实现js方法跳转的支持，依赖4组件

 

3. ### 一些全局配置 

   1. app.wpy中config配置全局引入的组件  

```json
usingComponents: {
      'van-icon': '/components/van/icon/index',
      'van-row': '/components/van/row/index',
      'van-col': '/components/van/col/index',
      'van-cell': '/components/van/cell/index',
      'van-cell-group': '/components/van/cell-group/index',
      'van-field': '/components/van/field/index',
      'van-button': '/components/van/button/index',
      'van-tag': '/components/van/tag/index',
      'van-grid': '/components/van/grid/index',
      'van-grid-item': '/components/van/grid-item/index'
},
```

2. app.wpy中style模块引入全局样式文件，其他页面就可以直接使用类名了，可以集合插件1 实现提示  

```css
@import url('/components/comm.less');
```

3. app.wpy中style模块，配置全局的盒子模型  

```css
view {
  -webkit-box-sizing: border-box;
  -moz-box-sizing: border-box;
  box-sizing: border-box;
}
```

