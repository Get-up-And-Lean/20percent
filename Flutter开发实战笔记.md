# Flutter开发实战笔记

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

