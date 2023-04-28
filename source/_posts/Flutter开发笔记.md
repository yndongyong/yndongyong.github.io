---
title: Flutter开发笔记
categories:
  - - flutter
tags:
  - - flutter
date: 2023-04-12 10:28:51
---



从Android切换到Flutter开发也有一段时间了，记录下使用过程的遇到的布局问题，时间长了是真的会忘。

## flutter widget分类

flutter 的widget很多，但是掌握常用的这些widget，就能完成80%的UI布局。

1. 基础widget：`Text`、`Button`、`Image`、`Icon`、`TextField`

2. 布局类widget：`Row` 、`Column`、`Expanded`、`Flex` 、`Stack`、`Positioned`、`Align`、`Center`、`Listview` 
3. 容器类widget：`Container`、`Scaffold`、`Padding`、`Center`

基础的`widget`，对应Android的View类相关的控件，布局类widget好比Android里的`ViewGroup`类相关的控件，`Row/Column`可以实现横向和纵向的线性布局，再复杂的布局都可以拆解为`Row/Column`实现。

## Widget常用技巧

### Container

`Container`是一个最常用的Widget，可以通过它设置宽高、背景颜色、圆角、阴影、边框、外边距、内边距，child的对齐方式等。它本身是一个组合widget，外边距、内边距、居中等都是通过嵌套对应的widget实现。

#### Container的组成

- 最里层是child元素
- 然后会被Align包裹
- 接着被Padding包裹，设置内边距
- 然后再被ColoredBox包裹
- 接着又是背景decoration包裹
- 再就是被前景foregroundDecoration包裹
- ConstrainedBox包裹
- 再被Padding包裹，设置margin外边距
- 最后被Transform包裹

#### Container 尺寸确定

1. 没有子节点时，Container会尝试去变得足够大，在父节点 的约束范围内

2. 带子节点的Container，会根据子节点尺寸调整自身尺寸，但如果Container设置了width、height以及constraints，则会按照这些参数来进行尺寸的调节。

#### Container布局行为

Container继承了StatelessWidget，由于组合了一系列的Widget，这些Widget都有自己的布局行为，因此导致Container的布局行为在不同widget组合下呈现比较复杂的布局行为。

> - 如果没有子节点、没有设置width、height以及constraints，并且父节点没有设置unbounded的限制，Container会将自身调整到足够小。
> - 如果没有子节点、对齐方式（alignment），但是提供了width、height或者constraints，那么Container会根据自身以及父节点的限制，将自身调节到足够小。
> - 如果没有子节点、width、height、constraints以及alignment，但是父节点提供了bounded限制，那么Container会按照父节点的限制，将自身调整到足够大。
> - 如果有alignment，父节点提供了unbounded限制，那么Container将会调节自身尺寸来包住child；
> - 如果有alignment，并且父节点提供了bounded限制，那么Container会将自身调整的足够大（在父节点的范围内），然后将child根据alignment调整位置；
> - 含有child，但是没有width、height、constraints以及alignment，Container会将父节点的constraints传递给child，并且根据child调整自身。
>
> 另外，margin以及padding属性也会影响到布局。

#### 设置宽高不起作用

给`Container`设置宽高不起作用，通常是`Container`的父widget给的`constraint`约束是紧约束，可以通过打断这种约束传递，达到设置宽高的目的。

1. 通过`UnconstrainedBox`  解除约束，让自身的约束变为无约束。
2. 通过`Center`、`Align`、`Row`、`Column`等组件放松约束，让自身的约束变为松约束。

### bottomSheet 的高度

#### 高度为自适应内容大小

通常情况下`bottomSheet`的高度为半屏，想要其高度为包裹内容的大小的话，可以指定`Column` `child`的

```dart
mainAxisSize: MainAxisSize.min
```

#### 高度为指定大小

可以给`root child`套个`Container`，指定`Container`的 `minHeight`

```dart
 constraints: minHeight != null ? BoxConstraints.tightForFinite(height: minHeight!) : null,
```

### bottomSheet适配双端

假如bottomSheet以及从贴着屏幕底部的dialog，其内部在底部有操作按钮的情况下，需要特殊处理一下，否则ios端可能存在小黑条遮在按钮上的情况。

```dart
var result = await Get.bottomSheet(
    SafeArea(
        child: child,
    ),
    //圆角
    shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.only(topRight: Radius.circular(8.w), topLeft: Radius.circular(8.w)),
    ),
    backgroundColor: Colors.white,
);
```



### Widget设置不重绘

PageView切换、或者长表单中上下滑动时，child widget会被回收重新绘制，期望不重绘的话，可以给child widget的`state` `mix` `AutomaticKeepAliveClientMixin` 同时重写 `wantKeepAlive` 方法，这样`widget`就不会重绘了。

```dart
class _TioFormFilePickerState extends State<TioFormFilePicker> with AutomaticKeepAliveClientMixin {

  @override
  bool get wantKeepAlive => true;

}  
```

### GestureDetector点击空白处不响应点击事件

GestureDetector默认行为是不响应空白位置的点击事件，会忽略不可见的元素，如果需要触发的话，可以修改`behavior`

```dart
GestureDetector(
    behavior: HitTestBehavior.translucent,//响应空白位置的点击
    onTap: () => xxxxx(),
    child: child,
);
```

### Text 行间距

Text的`TextStyle` 没有提供对应的设置文本行间距的方法，但是可以通过`height`属性设置行高，`height`默认为1，表示行高等于文字高度。假设将`height`设置为1.5，则表示行高为1.5倍的文字高度，因此行间距也会变高。

### TextFiled焦点处理

表单中有很多个`TextFiled`时，需要在这些`TextFiled`中切换焦点时，为每个TextFiled设置FocusNode，设置 FocusNode获取焦点，这样移动焦点的方式就比较繁琐，需要为每个TextFiled都要写一些模板代码。简洁一点的方式是可以为TextFiled设置`textInputAction：TextInputAction.next`，这样输入焦点就会顺着表单中的TextFiled自动移动。

ListView 滑动隐藏键盘

通过keyboardDismissBehavior属性：

```dart
keyboardDismissBehavior: ScrollViewKeyboardDismissBehavior.onDrag
```

### StatefulBuilder

StatelessWidget 组件内部 局部刷新，比如，RadioGroup ，Switch的切换,其内部的setState触发的刷新范围仅为widget内

```dart
StatefulBuilder(
  builder: (BuildContext context, StateSetter setState) {
    return RadioGroup.builder(
      activeColor: TyColors.colorSecondary,
      direction: Axis.horizontal,
      groupValue: value,
      onChanged: (newValue) {
        setState(() {
          value = newValue!;
        });
        if (null != newValue) {
          onChanged(newValue);
        }
      },
      items: options,
    );
  },
),
```



## FlutterEngineGroup 传参

flutter 混合工程中 原生侧的Flutter engine使用FlutterEngineGroup管理，跳转到Flutter的页面并且携带参数

### 原生侧

```kotlin
dartEntrypoint = DartExecutor.DartEntrypoint(
    FlutterInjector.instance().flutterLoader().findAppBundlePath(),
    "xxxxMain" //tag1 flutter 的main方法名称
)
val options = FlutterEngineGroup.Options(this)
options.dartEntrypoint = dartEntrypoint
  //options.initialRoute = path //和上一个同时设置会有两个跟页面
options.dartEntrypointArgs = arrayListOf("key1=value1&key2=value2")  //携带参数
flutterEngine = FlutterManager.flutterEngines.createAndRunEngine(options)
```

### flutter侧

```dart
@pragma('vm:entry-point')
void xxxxMain(args) async { //tag2
  //args 原生侧传递过来的参数，是个List<String>
  var params = Uri.splitQueryString(args[0]);
  runApp(const MyApp(initRoute: xxxxxx));
}
```

## Vertical（Horizontal） viewport was given unbounded height.

`ListView`没有被赋予一个确定的纵向约束，该错误通常是因为在`Column`（`Row`、`ListView`）中嵌套`ListView`时中出现，一般情况下`Column`给ListView传递的约束是loose constraint。有三种解决办法。

1. 在`ListView`外层嵌套一个`SizedBox`并设置`tight constraint`。
2. 在`ListView`外层嵌套一个`Expand`，撑开`Column`剩余可用空间。
3. 指定`ShrinkWrap`属性为true，让`ListView`能根据自身高度进行自适应，`listview`会收缩，缩小到其child的高度。

## 混合开发Fflutter页面的返回按钮处理

一个flutter表单详情页面，跳转的它的入口有两类，一类是原生侧直接跳转，此时该Flutter页面会被当做根路由；第二类是是正常的flutter路由跳转，对于这两种情况返回按钮的处理逻辑是不同的。第一种的情况下，点击返回按钮要求关闭flutter该页面；第二种情况下，点击返回按钮要求返回到上一个Flutter页面。可以判断当前路由能否被关闭，能的话关闭当前页面返回上一个，不能的话就是根路由使用MethodChannel关闭FlutterActivity容器。

```dart
ModalRoute.of(context)?.canPop ?? false
        ? Get.back()
        : schemeApi.closeFlutterPage(null);
```

### Factory单例

```
class SingletonTest{
  factory SingletonTest() => _instance ??= SingletonTest._();

  static SingletonTest? _instance;

  SingletonTest._();
}
```

使用`SingletonTest()`  getx中的依赖依赖注入相关的`GetInstance`就是这种写法