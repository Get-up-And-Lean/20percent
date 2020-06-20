# Flutter开发实战笔记

## TOC

[TOC]

## 下载

https://flutter.cn/docs/get-started/install/macos#get-sdk

## 配置环境变量

``export PATH="$PATH:[PATH_TO_FLUTTER_GIT_DIRECTORY]/flutter/bin"``

## 配置Flutter镜像源


```bash
export PUB_HOSTED_URL=https://pub.flutter-io.cn
export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn
```

最终的文件配置情况，看下图的24、25、29行：

![image-20200613210826514](https://gitee.com/jinxin.70/oss/raw/master/uPic/image-20200613210826514_2020_06_13_21_08_26.png)

## 修改SDK中的MAVEN_REPO

文件位置：

``/Users/akm/Downloads/app/flutter/packages/flutter_tools/gradle/flutter.gradle``

将``https://storage.googleapis.com/download.flutter.io``改成``https://storage.flutter-io.cn/download.flutter.io``



也就是替换``googleapis.com``为``flutter-io.cn``

![image-20200613211156902](https://gitee.com/jinxin.70/oss/raw/master/uPic/image-20200613211156902_2020_06_13_21_11_57.png)

参考：[Flutter中文网镜像源设置教程](https://flutter.cn/community/china)

## 安装 Android Studio

参考：https://flutter.cn/docs/get-started/install/macos#install-android-studio

## 配置 Android 模拟器

参考：https://flutter.cn/docs/get-started/install/macos#set-up-your-android-device

## 编辑工具设定之VS Code

参考：https://flutter.cn/docs/get-started/editor?tab=vscode

## 开发体验初探

参考：https://flutter.cn/docs/get-started/test-drive?tab=vscode

内容包括：

- 如何在VS code编辑器中创建应用

- 体验热加载

## 编写第一个 Flutter 应用

参考：https://flutter.cn/docs/get-started/codelab

功能：为一个创业公司生成建议的公司名称。用户可以选择和取消选择的名称、保存喜欢的名称。该代码一次生成十个名称，当用户滚动时，会生成一新批名称。

### 1. 创建初始化工程

关键内容：

- Material 风格的 widgets
- 主函数main
- StatelessWidget
- 在 Flutter 中，几乎所有都是 widget，包括对齐 (alignment)、填充 (padding) 和布局 (layout)。
- `Scaffold` 是 Material 库中提供的一个 widget，它提供了默认的导航栏、标题和包含主屏幕 widget 树的 body 属性。 widget 树可以很复杂。
- 一个 widget 的主要工作是提供一个 `build()` 方法来描述如何根据其他较低级别的 widgets 来显示自己。
- 本示例中的 body 的 widget 树中包含了一个 `Center` widget， Center widget 又包含一个 `Text` 子 widget， Center widget 可以将其子 widget 树对齐到屏幕中心。

### 2. 第二步：使用外部package

使用到的开源软件包：[english_words](https://pub.flutter-io.cn/packages/english_words) 

你可以在 [pub.dev](https://pub.flutter-io.cn/) 上找到 `english_words` 软件包以及其他许多开源软件包。

- 添加依赖包

`pubspec.yaml` 文件管理 Flutter 应用程序的 assets（资源，如图片、package等）。在pubspec.yaml 中，将 english_words（3.1.5 或更高版本）添加到依赖项列表，如下面高亮显示的行：

pubspec.yaml

             dependencies:            
                                   flutter:            
                                     sdk: flutter            
                                   cupertino_icons: ^0.1.2            
                    +              english_words: ^3.1.0
- 安装依赖包

```
flutter pub get
```

- 在 `lib/main.dart` 中引入

```dart
import 'package:flutter/material.dart';
import 'package:english_words/english_words.dart';
```

- 使用

```dart
class MyApp extends StatelessWidget {            
                                   @override            
                                   Widget build(BuildContext context) {            
                    +                final wordPair = WordPair.random();            
                                     return MaterialApp(            
                                       title: 'Welcome to Flutter',            
                                       home: Scaffold(            
        @@ -16,7 +18,7 @@    
                                           title: Text('Welcome to Flutter'),            
                                         ),            
                                         body: Center(            
                    -                      child: Text('Hello World'),            
                    +                      child: Text(wordPair.asPascalCase),            
                                         ),            
                                       ),            
                                     );
```

### 3. 第三步：添加一个 Stateful widget

State*less* widgets 是不可变的，这意味着它们的属性不能改变 —— 所有的值都是 final。

State*ful* widgets 持有的状态可能在 widget 生命周期中发生变化，实现一个 stateful widget 至少需要两个类： 1）一个 StatefulWidget 类；2）一个 State 类，StatefulWidget 类本身是不变的，但是 State 类在 widget 生命周期中始终存在。

在这一步，你将添加一个 stateful widget（有状态的 widget）—— `RandomWords`，它会创建自己的状态类 —— `RandomWordsState`，然后你需要将 `RandomWords` 内嵌到已有的无状态的 `MyApp` widget。

### 4. 第四步：创建一个无限滚动的 ListView

在这一步中，你将扩展（继承）`RandomWordsState` 类，以生成并显示单词对列表。当用户滚动时，`ListView` 中显示的列表将无限增长。 `ListView` 的 `builder` 工厂构造函数允许你按需建立一个懒加载的列表视图。

### 5. 以 profile 模式运行

关心性能，可以以 profile 模式运行。

