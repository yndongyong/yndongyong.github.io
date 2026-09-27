---
title: Dart 现代语法速查笔记（面向 Kotlin / Android 开发者）
categories:
  - - Flutter
  - - Dart
tags:
  - - Dart
  - - Flutter
  - - Kotlin
date: 2026-09-27 14:26:00
---

本文针对具备 Kotlin / Android 开发背景的同学，剔除琐碎细节与纯理论，聚焦 **Dart 2.18 → Dart 3.x** 在 Flutter 开发中最核心、最高频的现代语法特性与差异对照。

---

## 一、Kotlin 与 Dart 核心语法映射表

| 场景 / 概念 | Kotlin | Dart |
|---|---|---|
| 可变变量 | `var a = 1` | `var a = 1;` |
| 只读引用（运行时） | `val a = compute()` | `final a = compute();` |
| 编译期常量 | `const val A = 1` | `const a = 1;` |
| 延迟初始化 | `lateinit var a: String` | `late String a;` / `late final a = ...` |
| 可空类型 | `String?` | `String?` |
| 安全调用 | `user?.name` | `user?.name` |
| 空值合并 / 默认值 | `name ?: "Unknown"` | `name ?? 'Unknown'` |
| 空值赋值 | `if (a == null) a = b` | `a ??= b;` |
| 非空断言 | `name!!` | `name!` |
| 作用域变换 / 级联 | `obj.apply { ... }` | `obj..foo()..bar();` |
| Lambda / 箭头函数 | `{ x -> x * 2 }` | `(x) => x * 2` |
| 必填命名参数 | 构造器/函数命名调用 | `{required String name}` |
| 构造器参数自动绑定 | `class User(val name: String)` | `User(this.name);` |
| 继承父类参数透传 | `class Button(key: Key) : View(key)` | `Button({super.key});` |
| 接口声明 | `interface Repository` | `abstract interface class Repository` (Dart 3) |
| 密封类 | `sealed class UiState` | `sealed class UiState` (Dart 3) |
| 条件分支表达式 | `val x = when(state) { ... }` | `final x = switch(state) { ... };` (Dart 3) |
| 匿名数据结构 / 元组 | `Pair("A", 1)` | Record: `('A', 1)` (Dart 3) |
| 解构匹配 | `val (name, age) = user` | `final (name, age) = user;` (Dart 3) |
| 集合过滤 | `list.filter { it > 0 }` | `list.where((x) => x > 0).toList()` |
| 集合展开 | `val all = a + b` | `[...a, ...b]` |
| 异步单值 | `suspend fun load(): String` | `Future<String> load() async` |
| 异步事件流 | `Flow<Int>` / `flow { emit(1) }` | `Stream<int>` / `async* { yield 1; }` |
| 单例 / 伴生对象 | `object` / `companion object` | Top-level 顶层成员 / `static` / `factory` |
| 扩展函数 / 属性 | `val String.isEmail: Boolean` | `extension on String { bool get isEmail => ...; }` |

---

## 二、变量声明：var / final / const / late

```dart
// 1. var: 类型自动推断，变量可重新赋值
var name = 'Tom';
name = 'Jerry';

// 2. final: 运行时常量，仅可赋值一次（对应 Kotlin val）
final now = DateTime.now();

// 3. const: 编译期常量，用于 Flutter Widget 性能优化
const padding = EdgeInsets.all(16);
const widget = SizedBox(height: 10);

// 4. late: 延迟初始化（对应 Kotlin lateinit），不立即初始化但承诺使用前已赋值
late final TextEditingController controller;

// late 表达式同时具备类似 Kotlin lazy 的特性（第一次访问时求值）
late String heavyData = _computeHeavyData();
```

> **注意**：Flutter 中 Widget 属性若全部为 `const`，框架会复用编译期对象，避免频繁重绘。

---

## 三、空安全与运算符

Dart 默认开启健全空安全（Sound Null Safety）：

```dart
String? nullableStr; // 允许为 null

// 1. 安全调用 ?. 与空合并 ??（等价于 Kotlin ?:）
final length = nullableStr?.length ?? 0;

// 2. 空赋值操作符 ??=
nullableStr ??= 'Default Value'; // 为 null 时才赋值

// 3. 智能类型提升（Smart Cast）
if (nullableStr != null) {
  print(nullableStr.length); // 自动提升为非空 String，无需手动断言
}

// 4. 慎用非空断言 !
print(nullableStr!); // 等价于 Kotlin 的 !!

// 5. 类型判断与转换
if (value is String) { ... }
final str = value as String;

// 6. 整除运算符 ~/（容易忽略的 Dart 特有操作符）
final result = 10 ~/ 3; // 结果为整数 3
```

---

## 四、函数与 Flutter 参数风格

Dart 语法深度适配了 Flutter 嵌套组件树的设计：

### 1. 命名参数与必选标记 `required`

```dart
// 使用 {} 包裹即为命名参数，配合 required 实现严格约束
void login({
  required String username,
  required String password,
  bool rememberMe = true, // 默认参数
}) {}

// 调用端具备极高可读性
login(
  username: 'tom',
  password: '123',
);
```

### 2. 回调函数与级联操作符 `..`

```dart
// VoidCallback 对应 Kotlin 的 () -> Unit
Widget buildButton({required VoidCallback onPressed}) {
  return ElevatedButton(onPressed: onPressed, child: const Text('OK'));
}

// 级联操作符 ..（类似 Kotlin 的 apply，链式操作同一个对象）
final controller = TextEditingController()
  ..text = 'Hello'
  ..selection = const TextSelection.collapsed(offset: 5);
```

---

## 五、面向对象与现代化类设计

### 1. 构造函数简化与 `super.key`

```dart
class UserWidget extends StatelessWidget {
  // super.key 替代了旧版的 : super(key: key)
  const UserWidget({
    super.key,
    required this.name,
    this.age = 18,
  });

  final String name;
  final int age;

  @override
  Widget build(BuildContext context) => Text('$name - $age');
}
```

### 2. 命名构造函数与 `factory`

```dart
class User {
  final String name;
  User(this.name);

  // 命名构造函数（替代 Kotlin companion object 工厂方法）
  User.guest() : name = 'Guest';

  // factory 构造函数：常用于 JSON 反序列化、缓存或根据条件返回子类
  factory User.fromJson(Map<String, dynamic> json) {
    return User(json['name'] as String);
  }
}
```

### 3. 私有作用域、Mixin 与接口修饰符（Dart 3）

- **私有标识**：变量或方法前加下划线 `_name`，其私有范围是 **文件/Library 级**（而非仅仅 Class 内）。
- **`mixin`**：实现跨类行为复用（`class Controller with LogMixin {}`）。
- **`abstract interface class`**：明确声明纯接口契约。
- **`extension`**：扩展函数/Getter：
  ```dart
  extension StringExt on String {
    bool get isEmail => contains('@');
  }
  ```

---

## 六、集合与内嵌语法（Collection if / for / spread）

这是 Flutter 声明式 UI 编程的核心利器：

```dart
final bool isLoggedIn = true;
final items = ['Item 1', 'Item 2'];

final widgets = [
  const HeaderWidget(),

  // 1. Collection if：满足条件才插入元素
  if (isLoggedIn) 
    const UserProfileWidget()
  else 
    const LoginButton(),

  // 2. Collection for：直接展开循环生成组件
  for (final item in items) 
    ListTile(title: Text(item)),

  // 3. 展开操作符 ... 与空安全展开 ...?
  ...extraWidgets,
  ...?nullableWidgets, // 若为 null 则静默忽略
];
```

### 常见集合流式 API 与 Kotlin 对照

```dart
// 过滤：where 对应 Kotlin 的 filter
final activeUsers = users.where((u) => u.isActive).toList();

// 变换：map 对应 Kotlin 的 map
final names = users.map((u) => u.name).toList();

// 聚合：fold / reduce
final sum = numbers.fold(0, (prev, elem) => prev + elem);

// 判断：any / every 对应 Kotlin 的 any / all
final hasAdmin = users.any((u) => u.isAdmin);
```

---

## 七、Dart 3 核心：Record、Pattern 与 Switch 表达式

Dart 3 引入了完整的模式匹配与元组支持，极大地重构了状态管理逻辑。

### 1. 记录（Record）：轻量级匿名数据返回

```dart
// 多返回值无需专门定义 DTO Class，支持位置与命名参数
(String token, int expiresIn) login() {
  return ('token_xxx', 3600);
}

// 模式匹配解构
final (token, expire) = login();
```

### 2. `sealed class` 配合 `switch` 表达式建模 UI 状态

类似 Kotlin 的 Sealed Class + `when`：

```dart
// 1. 定义密封类状态
sealed class UiState {}
class Loading extends UiState {}
class Success extends UiState {
  final List<String> data;
  Success(this.data);
}
class Error extends UiState {
  final String message;
  Error(this.message);
}

// 2. 现代 Switch 表达式模式匹配解构（详尽性检查，少写 case 编译报错）
Widget buildBody(UiState state) {
  return switch (state) {
    Loading() => const CircularProgressIndicator(),
    Success(:final data) => ListView(children: [for (final item in data) Text(item)]),
    Error(:final message) => Text('Error: $message'),
  };
}
```

---

## 八、异步编程：Future 与 Stream

| 概念 | 核心语法 | 对齐 Kotlin |
|---|---|---|
| 单次异步任务 | `Future<T>` + `async / await` | `suspend fun` 挂起函数 |
| 异步并发执行 | `await Future.wait([taskA(), taskB()])` | `coroutineScope { awaitAll(...) }` |
| 异步事件流 | `Stream<T>` + `async*` / `yield` | `Flow<T>` + `flow { emit(...) }` |
| 消费事件流 | `await for (final item in stream)` | `flow.collect { item -> ... }` |

```dart
// 1. Future 异步请求与错误重抛
Future<User> fetchUser(String id) async {
  try {
    final response = await http.get('/user/$id');
    return User.fromJson(response.data);
  } catch (e) {
    logError(e);
    rethrow; // 重新向上抛出原始异常
  }
}

// 2. Stream 数据生成器（对应 Kotlin flow 构建器）
Stream<int> countDown(int from) async* {
  for (int i = from; i >= 0; i--) {
    await Future.delayed(const Duration(seconds: 1));
    yield i; // 产生数据流
  }
}
```
