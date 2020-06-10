# 掌握Gradle构建工具

## 学习动机：

目前开发的项目使用的构建工具是gradle，最初搭建项目骨架的时候是参照已有gradle项目配置搭建的，遇到问题基本上是靠搜索引擎搜索，对gradle的认知仅仅停留在会使用，想要进一步提高，或者有一个成体系的认知。

## 学习目的：

## 正文

### 目前主流构建工具

![](https://gitee.com/jinxin.70/oss/raw/master/uPic/DjoPSp_2020_01_31_10_44_23.png)

### Gradle是什么

一个开源的项目自动化构建工具，建立在Apache Ant和Apache Maven概念的基础上，并引入了基于Groovy的特定领域语言（DSL），而不是使用XML形式管理构建脚本。

## 学习安排

### 快速尝鲜

- 准备使用Gradle
- 第一个Gradle项目

### 基本原理

- 构建脚本介绍
- 依赖管理

### 深入实战

- 多项目构建
- 测试
- 发布

## 准备使用Gradle

### 安装

- 安装和配置好JDK
- 从官网下载Gradle

- 配置环境变量，``GRADLE_HOME``
- 添加到path，``%GRADLE_HOME%\bin``
- 验证是否安装成功，``gradle -v``

## Groovy基础知识

### Groovy是什么

Groovy是用于Java虚拟机的一种敏捷的动态语言，它是一种成熟的面向对象编程语言，既可以用于面向对象编程，又可以用作纯粹的脚本语言。使用该种语言不必编写过多的代码，同时又具有闭包和动态语言中的其他特性。

### 与Java比较

- 完全兼容Java的语法
- 分号是可选的
- 类、方法默认是public的
- 编译器给属性自动添加getter/setter方法
- 属性可以直接用点号获取
- 最后一个表达式的值会被作为返回值
- ``==``等同于``equals()``，不会有``NullPointerExceptions``

### 高效的Groovy特性

- ``assert``语句
- 可选类型定义
- 可选的括号
- 字符串
- 集合API
- 闭包

## 使用IDEA创建Gradle项目

### 1. New Project

![](https://gitee.com/jinxin.70/oss/raw/master/uPic/image-20200131112158455_2020_01_31_11_45_57.png)

### 2. 写上GroupId和ArtifactId

![](https://gitee.com/jinxin.70/oss/raw/master/uPic/e0pjDS_2020_01_31_11_24_27.png)

### 3. 填写Project Name，点击Finish完成

![](https://gitee.com/jinxin.70/oss/raw/master/uPic/image-20200131112517494_2020_01_31_11_46_37.png)

### 4. 打开Groovy Console

<img src="https://gitee.com/jinxin.70/oss/raw/master/uPic/pIMGJg_2020_01_31_11_26_45.png" style="zoom:50%;" />

### 5. 验证Groovy语法

```groovy
public class ProjectVersion{
    private int major
    private int minor

    ProjectVersion(int major, int minor) {
        this.major = major
        this.minor = minor
    }
}
ProjectVersion v1 = new ProjectVersion(1,1)
println v1.minor

ProjectVersion v2 = null;
v2 == v1
```

<img src="https://gitee.com/jinxin.70/oss/raw/master/uPic/IssJRd_2020_01_31_11_30_55.png" style="zoom:50%;" />

执行结果：

<img src="https://gitee.com/jinxin.70/oss/raw/master/uPic/nrbJYM_2020_01_31_11_32_09.png" style="zoom:50%;" />

上面可以看到``v2==v1``的表达式返回``false``，其实调用的是``Object``类的``equals``方法。

## Groovy高级特性实操

```groovy
def version = 1
assert version == 1
println version

//字符串
def s1 = 'Eucaly'
def s2 = "gradle version is ${version}"
def s3 = '''my
name
is
Eucaly'''

println(s1)
println(s2)
println(s3)

//集合api
def buildTools = ['ant','maven']
buildTools << 'gradle'
assert buildTools.getClass() == ArrayList
assert buildTools.size() == 3

def buildYears = ['ant':2000,'maven':'2004']
buildYears.gradle = 2009

println(buildYears.ant)
println(buildYears['gradle'])
println(buildYears.getClass())
println(buildYears)

//闭包,经常被当作方法参数使用
def c1 = {
    v -> print v
}

def c2 = {
    print 'hello'
}

def method1(Closure closure){
    closure('param')
}

def method2(Closure closure){
    closure()
}

method1(c1)
method2(c2)
```

## 读懂Groovy构建脚本

```groovy
apply plugin:'java'

version ='0.1'

repositories {
    mavenCentral()
}

dependencies {
    compile 'common-codec:commons-condec:1.6'
}
```

上面的脚本代码通常是项目中build.gradle文件中的构建代码模板，下面从groovy脚本的角度来一句句分析是什么意思。

1. **构建脚步中默认都有一个Project实例，脚本中默认的作用域都是Project**

2. ``apply plugin:'java'``

   plugin:'java'是命名参数，意思是plugin参数的值是java；

   apply是Project实例中的方法,这里调用apply方法，省略了括号。

3. ``version ='0.1'``

   version是Project实例中的一个属性

4. ``repositories {
       mavenCentral()
   }``

  repositories也是Project实例中的方法，入参是一个闭包；
  整个花括号部分代码块``{mavenCentral()}``是一个闭包。
  
5. ``dependencies {
      compile 'common-codec:commons-condec:1.6'
  }``

  dependencies也是Project实例中的方法，入参是一个闭包

