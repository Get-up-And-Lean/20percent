# 2/41为什么要掌握 Flutter？

移动开发技术手段从原始的原生应用开发，到 Google 和 Apple 各自推出新的语言 Kotiln、Swift，然后又发展到各种跨平台Hybrid App 开发，如 React Native、Weex、小程序等等。不过这些方案或多或少都有一些局限性和缺点，于是 Google 推出了新的跨平台移动应用开发技术：Flutter。

为什么说 Flutter 将是未来的主流跨平台开发框架？

Google 的 Flutter 开发应用的体验和流畅度基本和原生体验一致，感觉不到不流畅和卡顿。

我们知道在移动平台上，原生应用的体验最好、流畅度最高、性能也最好。而目前的跨平台技术和框架的流畅度和体验远远达不到原生的体验，多少都会卡顿和丢帧，但是 Google 官方说 Flutter 可以达到120 FPS。

Flutter 最出色的地方就是自建了绘制引擎，使得跨平台开发一套代码可以创造出和原生应用相同的体验。并且 Flutter 开发效率非常高，SDK 里所有的布局、控件都组件化，采用 React 方式。

**Flutter 的开发不仅仅局限于移动跨平台，目前已经支持 Web 开发、后端开发、PC 桌面应用开发（内测中）、嵌入式开发（内测中）。** 这也是 Flutter 变得越来越受关注，越来越多大公司和开发者进行使用的原因之一。

Flutter 支持多种开发工具的插件化使用，这就大大的丰富了各种开发工具可以进行 Flutter 的开发和调试，也满足了不同开发者的开发习惯。

同时 Flutter 不但做到了一套代码逻辑实现 Android 和 iOS 平台的跨平台运行，而且无需像 React Native 等技术那样，部分和原生交互的逻辑需要写两套代码逻辑，Flutter 只需写一套代码，大部分功能官方 SDK 里已经支持，并在不断更新拓展。而且如果需要一些与原生交互的部分，都是通过插件化形式使用，依然是一套代码逻辑多平台兼容。后面将会详细讲解。

接下来，我们就开始我们的认识 Flutter 技术之旅吧。本文将主要介绍：

> - Flutter 的诞生
> - 目前各种跨平台方案的对比
> - Flutter 特点
> - Flutter 框架结构
> - Flutter社区活跃度和趋势

### 1 Flutter 的诞生

![Flutter 发布](https://images.gitbook.cn/Ft_FAvqblB9kYBS4NKMk482KmT5f) ( 图片来自 Google 开发者官方 )

Flutter是 Google 于 2015 年 5 月 3 日推出的免费开源跨平台开发框架，可以快速开发 Android 和 iOS 应用，同时也将是未来的 Google Fuchsia OS 下开发应用的主要技术框架。**未来的 Flutter 的开发不仅仅局限于移动跨平台，目前已经支持 Web 开发、后端开发、PC 桌面应用开发（内测中）、嵌入式开发（内测中）。**

Flutter 的第一个版本 SDK 运行在 Android 操作系统上，被称作“Sky”。第一个开发者版本在 2015 年的 Flutter 开发者会议上被公布，并且对外宣称 Flutter 的开发的应用目标为实现 120FPS的渲染性能，原生的渲染性能一般为 80FPS。由于之前一直是开发版本，所以 Flutter 不温不火，只有少数的一些公司和 Google 内部在尝试使用。

终于在 2018 年 12 月 5 日，Google Flutter 团队正式发布了 Flutter 1.0 正式版，正式版的发布意味着 Flutter 在经过开发、测试、应用上已经基本上稳定和满足大部分开发需求了。这也使得Flutter 在发布正式版后，更多的大公司、开发者纷纷转型使用和学习 Flutter 进行跨平台应用的开发。

目前在 Google 内部，Flutter 已经被广泛用于多个产品，比如 Google Ads 已经将其产品的 iOS 版本和 Android 版本转向使用 Flutter。在正式版本之前，全世界已经有多个公司开始使用 Flutter 来开发应用，包括 Abbey Road Studios、阿里巴巴、Capital One、Groupon、Hamilton、京东、Philips Hue、Reflectly 以及腾讯等。而正式版的功能基本上已经比较完善，其他的扩展更强大的功能 Google 官方也正在规划扩展开发。目前 Flutter 1.5 稳定版已经于 2019 年 5 月 8 号发布，这样的更新频率，给更多的开发者和公司增加了动力。Flutter 势必也将成为未来的跨平台开发主流趋势！

接下来回顾下 2018 年和 2019 年 Flutter 的发展情况：

- 2 月底在世界移动大会 （MWC）上宣布了第一个 Beta 版发布;
- 5 月的 Google I/O 大会上发布了 Beta 3;
- 6 月底的 GMTC 宣布了首个发布预览版;
- 9 月的谷歌开发者大会（Google Developer Days）上，发布预览版 2。
- 12 月宣布发布正式稳定 1.0 版;
- 2019年 2 月宣布发布稳定版 1.2 版本 SDK。
- 2019年 5 月宣布发布稳定版 1.5 版本 SDK。

作为 Flutter 1.0 之后的首次更新， Flutter SDK 1.2 稳定版围绕以下几点进行了重点优化与改进：

- 提升核心框架的稳定性、性能和质量；
- 改进现有 Widget 视觉效果和功能；
- 为 Flutter 开发者提供全新的基于 Web 的调试工具。

![Flutter 分支](https://images.gitbook.cn/FsrGs2aBhH4xCRBJmlkEcJa-rfKp)

目前 Flutter 的社区非常活跃，Flutter 在 Github 最受欢迎的开源软件中排名前 50，国内也有大量的开发者开始使用 Flutter 构建跨平台 (Android & iOS) 的应用，如：阿里巴巴、腾讯、京东等都使用 Flutter 发布了自己的应用。

Google 官方 Flutter 团队计划，未来也将支持 Flutter Web 和 Flutter PC 的应用移植开发，让我们拭目以待吧！

### 2 目前各种跨平台方案的对比

目前我们在开发应用时，需要同时兼容 iOS 和 Android 两种平台时有两种技术选择：走原生开发路线，把界面和逻辑在不同平台分别实现；抑或用同一套代码兼容多个平台，但这往往意味着运行速度和产品体验的损失。除了原生外，目前跨平台技术一般是混合开发，如采用 H5、React Native、Weex、小程序等技术实现跨平台应用。不过这些或多或少都能感觉到卡顿和体验不流畅，并且开发和学习成本非常高，而且都有各自的局限性。

Flutter 的出现就是为我们提供了一套两全其美的解决方案：既能用原生代码直接调用的方式来加速图形渲染和 UI 绘制，又能同时运行在两大主流移动操作系统上，并且体验和流畅度和原生基本一致、开发效率也非常高、学习难度和成本低。

那么接下来看下几种方案的对比情况：

| 技术           |             性能 |      开发效率      |      渲染方式       |               学习成本               |          可扩展性          |
| :------------- | ---------------: | :----------------: | :-----------------: | :----------------------------------: | :------------------------: |
| Flutter        | 高，接近原生体验 |         高         | Skia 高性能自绘引擎 |          低，Widget 组件化           | 高，采用插件化的库进行扩展 |
| RN/Weex/小程序 |     有延迟，一般 | 一般，复杂、效率低 |   JS 驱动原生渲染   |               高，复杂               |            一般            |
| 原生应用       |               高 |        一般        |      原生渲染       | 高，需要学习 Android 和 iOS 原生 API |             高             |

从上面的对比可以看出，**Flutter 的优势明显：高体验度、高开发效率、低学习成本、高可扩展性，未来 Google Flutter 团队还将使 Flutter 支持 PC 和 Web 的跨平台开发，真正全平台。** 在 Flutter 1.0 正式版本尚未推出之前，已经有成百上千的基于 Flutter 开发的应用在 Apple Store 和 Google Play 上架，相信 Flutter 将会被越来越多的开发者和公司所接受和采用。

### 3 Flutter 特点

**Flutter 是一个框架，基于 Dart 语言编写，语言风格和 React 很像。Flutter 里几乎都是采用组件的形式进行构建应用和功能的，组件采用现代响应式框架构建，中心思想是用组件（Widget）构建你的 UI。**

一切对象都是组件，Flutter 可以说是全新的技术和平台框架，学习和开发起来并不难，开发应用的效率也相对于原生提升了很多，并且运行速度和原生几乎没太大差别，远远高于 HTML 的应用的渲染速度。Flutter 的引擎使用 C++ 开发，基础库由 Dart 编写，提供了用 Flutter 构建应用所需的基本的类和函数。

我们看下 Flutter 官方的架构图：

![Flutter 官方架构图](https://images.gitbook.cn/FgQtZgc6ELmrZFV9zdtxjZ34ueCS) ( Flutter 官方架构图 - 来自 Flutter 官方网站 )

可以看出核心引擎是使用 C++ 编写，上层采用 Dart 语言编写的 SDK，采用 React 风格组件化方式。并且提供了 Material 和 Cupertino 两套风格的 UI Widget，使用起来非常方便。

那么接下来我们详细了解下 Flutter 的特点：

- Flutter 的一个重要的特点就是速度快。它基于 Skia 2D 硬件加速图形引擎，该引擎也同样用在了 Chrome 和 Android 平台。媲美原生应用的速度，让用户体验和流畅度做到更好。Flutter 的代码基于 Dart 平台，可以被编译成 iOS 和 Android 平台上 32 位和 64 位的 ARM 代码。
- Flutter 非常高效。Flutter 引入了 Stateful Hot Reload（保持应用状态的热重载），这个革命性的新特性可以让移动开发者和设计师们时时快速预览应用程序。通过 Stateful Hot Reload，无需重新启动应用，就可以在程序运行的时候直接看到代码修改之后的效果。在Flutter官方的用户反馈中，很多开发者表示这个特性使得开发效率提升了三倍以上。
- Flutter 是开放的、开源的。Flutter 是一个基于 BSD-style 许可的开源项目，全球数百位开发者在为Flutter贡献代码。Flutter 的插件生态系统平台也已经非常的丰富，有数千款插件已经发布，避免了重复造轮子。由于 Flutter 应用程序使用标准的 Android 和 iOS 的编译打包工具 (build tools)，因此它的开放性还体现在我们可以使用原生开发资源和技术。比如，我们依然可以在 Android 上使用 Kotlin 或者 Java，在 IOS 上使用 Swift 或者 Objective-C 来写逻辑或者界面，使得可以Flutter和原生混合开发。
- Flutter 提供了两套 UI Widget 风格库：Material 和 Cupertino，这使得我们可以方便的快速构建 Android 和 iOS 不同平台风格的应用，大大提升了开发效率。组件化开发风格也使得绘制UI的效率大大提升，学习成本也降低了很多。
- Flutter 支持多种开发工具进行开发，比如 Visual Studio Code、Android Studio、IntelliJ 或其他开发工具，只需要安装相关的插件即可。

![Visual Studio Code 预览](https://images.gitbook.cn/Fl5aBUEHmI92ENafsWi8f5SE2G1H)

Google Flutter 团队官方也宣布，Flutter Web SDK（Hummingbird）已经在研发中，我们先来简单的看下其架构：

![Flutter Web SDK（Hummingbird）](https://images.gitbook.cn/FufKtpCKmttZnmx1WyYKaqMxSFg3)

目前在 Google I/O 2019 大会当天 Flutter Web SDK 已经正式发布，而且开发出的 Web 性能非常高。

### 4 Flutter 社区

看一个技术和语言的发展情况和支持情况看它的相关资源、社区等也非常重要。Flutter 技术已经被很多大公司采用，具体案例列表，可以在官方查看：[flutter.dev/showcase](https://flutter.dev/showcase) 或[https://itsallwidgets.com](https://itsallwidgets.com/)。

![ShowCase](https://images.gitbook.cn/Fq8ICBTBd_RiwQUhtkG4iV2RGpnL) （来自 Flutter 官方：https://flutter.dev/showcase ）

我们看下 Flutter 官方 GitHub 的更新情况:

![Flutter 官方 Github](https://images.gitbook.cn/Fsa-vCUhR31-tM7SCghhldwKJ-JA)

**开发者平时可以关注 Github 的更新动态，官方更新频率也是很快的，这点非常好，也令关注者和使用者对 Flutter 的未来更加有信心，因为社区和生态、更新频率对开发者非常的有帮助。** 遇到问题除了使用搜索引擎搜索外，也可以在官方 GitHub 的 Issues 里进行搜索或者提问。

我们再看下官方版本的更新频率：

![Flutter官方 Github](https://images.gitbook.cn/FkNVhza1yO0Ticqs_zCxbxVdA-gk)

![Flutter官方 Github](https://images.gitbook.cn/FhstoC3_fX1vDEAK1lu5RTy2pcWE)

除了这些以外，我们还可以在官方的仓库进行查找第三方插件库进行使用：https://pub.dartlang.org/。

![Flutter 官方 Pub](https://images.gitbook.cn/FngcfvA4CZOQdwNuXYU11ziV-u7k)

里面有非常多的插件库供我们使用，还配备有相关文档，当然我们也可以提交自己的开源插件库到上面。

![Flutter 官方 Pub](https://images.gitbook.cn/FtT0Nzy0g7ibGI6cVZdxsm401o-O)

### 5 Flutter 未来规划

就目前而言，Flutter 的首要目标平台是 iOS 和 Android，但 Flutter官方团队也在不断探索将 Flutter 拓展到手机端以外的更多平台上，如 Web、PC、嵌入式等平台。实现真正的跨平台：一套代码规范多平台运行。

事实上，Flutter 的设计理念就是希望它可以作为一个灵活且便携的 UI 工具包，以适应各种需要绘制屏幕内容的平台。

其中 Flutter 的一些进展已经公布，Flutter Desktop Embedding（[google/flutter-desktop-embedding](https://github.com/google/flutter-desktop-embedding) ）就是其中的一个，这是一个使 Flutter 运行于 macOS、Linux 和 Windows 等桌面操作系统的项目。前不久，官方尝试在树莓派平台运行了 Flutter 应用，以非正式和探索的形式向用户展示 Flutter 是有可能运行在一些没有完整桌面环境的小型设备中的。

除了这些，还有 Flutter Web（Hummingbird）。Hummingbird 是一个基于 Web 实现的 Flutter 运行时环境。它利用了 Dart 语言能被编译成 JavaScript 的特性。这个项目让 Flutter 应用程序能够无需改动就运行在标准 Web 平台，目前还在开发测试中：

![Hummingbird](https://images.gitbook.cn/FmTf48onc3QRIruSx_sTGLYL2UCw)

Flutter 团队也于 2019 年 1 月 27日发布了 2019 年 Flutter 规划路线。

以下几点 Flutter 今年会着重关注:

- 核心和基础
- 易用性
- 生态系统
- 移动端之外的支持
- 动态更新
- 工具链

当然，我们也可以提一些反馈给官方：

- 通过 Issues：[flutter/issues/new/choose](https://github.com/flutter/flutter/issues/new/choose)
- 邮件群组：[flutter-dev](https://groups.google.com/forum/#!forum/flutter-dev)

Flutter 目前有四个版本: master、dev、beta 和 stable，质量和稳定性从前向后依次递增，发布速度当然也会是依次相对放缓。

官方计划每个月发布一个 beta 版本，这个发布通常会是在月初，全年会发布四个较大的正式 (stable) 版本。在生产环境里，还是建议大家使用 Flutter 的正式版本。

这个是官方的版本发布流程：[flutter/wiki/Release-process](https://github.com/flutter/flutter/wiki/Release-process)

如果大家对 Flutter 每个月将会发布什么感兴趣的话，可以在官方 GitHub 上的 milestones 页面查看：[flutter/milestones](https://github.com/flutter/flutter/milestones?direction=asc&sort=due_date&state=open)

![Flutter Future](https://images.gitbook.cn/FsYt-ZDHhlNMxvFwElxsSZom95e0) （ 图片来自 Google 开发者官方 ）

### 总结

Flutter 现已进入 GitHub Top 20 软件库，通过这门课程希望大家可以对 Flutter 进一步了解，并且对它充满信心，也希望大家有所收获。

Flutter 成为未来主流跨平台开发框架技术已经势在必行，它开发高效、性能优秀、更新频率快、插件三方库支持多、Google 团队的技术支持给力、一套代码多终端运行，这些都非常的吸引人。

大家可以：

- 去官方 GitHub 查看官方动态，或者去 Flutter Pub 查看下仓库的使用方法，以便对 Flutter 有更加深入的了解。
- 熟悉 Flutter 特点和未来趋势，做好后续课程的学习和开发的准备。

入门 Flutter，掌握未来技术主流的主动权！