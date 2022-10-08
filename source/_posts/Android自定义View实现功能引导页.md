---
title: Android自定义View实现功能引导页
date: 2018-08-01 16:43:28
categories: 
- android
tags: 
- 自定义View
---


### [库地址](https://github.com/yndongyong/GuideView)

[demo](https://github.com/yndongyong/GuideView/blob/master/dest/app_guideview-debug.apk)

### 作用
实现页面引导，提示用户操作。用户引导结合场景，以图层的形式叠加到对应的View上。高亮效果支持矩形、圆角矩形、圆形、椭圆四种形状，以及支持高斯模糊的效果。

效果预览图

![功能引导](http://oav23hfp9.bkt.clouddn.com/18-8-1/91135939.jpg)

### 特性

1. 使用自定义view实现，显示时机控制为界面绘制完成后的下一个帧
2. 以图层的形式叠加到UI控件上
3. 高亮效果支持矩形、圆角矩形、圆形、椭圆四种
4. 支持高斯模糊效果
5. 提示布局方向支持左、上、右、下四个方向的居中，支持扩展
6. 支持全部一起显示，或是一个接一个的显示方式


### changelog

**v0.0.1**

实现效果，可用

### 思路

0. 收集要高亮显示的view的左上点坐标、宽高信息
1. 取到Activity的decorview，添加一层FrameLayout布局
2. 自定义view(GuideView)中根据高亮显示的view的坐标宽高信息，使用PorterDuffXfermode 的Mode.XOR，画出镂空效果，同一个坐标上正常的画一次，Mode.XOR的模式绘制第二次即可以实现镂空的效果
3. 在GuideView中根据高亮view的坐标 addView（功能提示的view），代码中处理了addView 以及requestLayout之后布局会闪烁的问题。

PorterDuffXfermode 的Mode.XOR 原理参见：

![图像合成](https://upload-images.jianshu.io/upload_images/2041548-d964105abf4be5d9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/312)

### 使用方式

```
new GuideViewHelper.Builder()
      .addView(id_switch_1, new BottomCenterItemDecoration(R.layout.item_decoration_0))
      .padding(5)
      .setHighLightShape(HighLightShape.TYPE_CIRCULAR)
      .showAll(true)
      .build()
      .show(this);
                
```

> 注：支持直接onCreate方法中调用show方法

#### 1、提供的对齐方式

提示布局方向支持左、上、右、下四个方向上的居中对齐显示

显示在左边与高亮view在水平方向居中对齐

``` LeftCenterItemDecoration ```

显示在上方与高亮view在垂直方向居中对齐

``` TopCenterItemDecoration ```

显示在右方与高亮view在水平方向居中对齐

``` RightCenterItemDecoration ```


显示在下方与高亮view在垂直方向居中对齐

``` BottomCenterItemDecoration ```

#### 2、 扩展对齐方式

提供的对齐方式若不满足需求，可以继承ItemDecoration扩展，实现如下的方法：

``` public abstract int[] getOffsetLeftAndTop(CutoutViewInfo cutoutViewInfo, int offsetX, int offsetY); ```

该方法返回一个int[] 数组，存放功能提示view的left 和 top 坐标

#### 5、暴露的回调方法 OnGuideViewDismissListener

回调时机：当功能提示view全部显示完关闭时

 ```
 guideViewHelper.show(MainActivity.this, new GuideView.OnGuideViewDismissListener() {
                    @Override
                    public void onDisMiss() {
                        Toast.makeText(MainActivity.this, "guide view dismiss", Toast.LENGTH_SHORT).show();
                    }
                });
 
 ```

