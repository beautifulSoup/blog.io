---
layout: post
title: Android持续集成以及测试覆盖率可视化
date: 2016-4-12
categories: blog
author: beautifulSoup
tags: [Development,Android]
description: 本篇博客主要介绍如何使用Travis CI进行持续集成。使用Jacoco生成测试覆盖率报告，并使用Codecv实现测试覆盖率的可视化图标。

---

## 背景

很多开源项目在README中会有几个小图标来表示build情况，测试覆盖率等。如
![](https://img.alicdn.com/imgextra/i3/754328530/TB2sF_JmVXXXXbQXpXXXXXXXXXX_!!754328530.png)

看起来感觉很牛逼的样子，其实实现起来很简单，只需几步，就能让你的开源项目也变得牛逼起来。

## Travis-CI

Travis-CI是一款持续集成工具，对开源项目免费。免除了Jenkins搭建服务器的工作。用户只要完成以下简单的几步就能接入Travis。

1. 通过Github账号登录https://travis-ci.org/。   
2. 在项目根目录添加.travis.yml 文件。
3. git add -> commit -> push.

之后再每次push之后Travis-CI就会根据.travis.yml对项目进行build。然后就可以在Travis网站控制台上查看build的情况。在build完成之后Travis也会通过邮件的方式通知你。

在这三步中，最麻烦的一步就是.travis.yml的创建。具体文档可以参考官网。以下给出我的项目的yml文件作为例子分步分析。
完整文档：

```
language: android
jdk:
    - oraclejdk8
env:
  matrix:
    - ANDROID_TARGET=android-21 ANDROID_ABI=armeabi-v7a
  global:
    - ADB_INSTALL_TIMEOUT=8
android:
  components:
    - tools
    - platform-tools
    - build-tools-23.0.2
    - android-23
    - extra-android-m2repository
    - extra-android-support
    - sys-img-armeabi-v7a-android-21
before_script:
    - echo no | android create avd --force -n test -t $ANDROID_TARGET --abi $ANDROID_ABI
    - emulator -avd test -no-skin -no-audio -no-window &
    - android-wait-for-emulator
    - adb shell input keyevent 82 &
script:
    - ./gradlew :lib:createDebugAndroidTestCoverageReport --info --stacktrace
after_success:
    - bash <(curl -s https://codecov.io/bash)

```

告诉Travis-CI项目语言。这是必要的一步。

```
language: android
```

设置JDK版本。非必要，但建议写上。

```
jdk:
    - oraclejdk8
```

设置环境变量

```
env:
  matrix:
    - ANDROID_TARGET=android-21 ANDROID_ABI=armeabi-v7a
  global:
    - ADB_INSTALL_TIMEOUT=8
```
matrix下面的配置会应用到当前build，而global应用到所有builds。matrix 下两项设置了TARGET api以及ABI，注意需要与Gradle中配置一致。global下这项设置了虚拟机安装apk的超时时间，如果你的项目不需要Instrument Test，可以不用加，但是如果有，且在持续集成中需要测试则需要加上，因为默认情况下安装超时时间为2分钟，某些情况下可能因为没有在2分钟内安装成功而导致集成失败。

设置项目构建依赖

```
android:
  components:
    - tools
    - platform-tools
    - build-tools-23.0.2
    - android-23
    - extra-android-m2repository
    - extra-android-support
    - sys-img-armeabi-v7a-android-21
```

tools platform-tools 这两项表示该次build会使用最新的SDK tools, 这两项最好加上，因为如果没有，可能会报错说找不到指定版本的sdk tools。后面是对build-tools和compile sdk version的设置，注意与gradle中一致。如果你的项目中依赖了AppCompat和design support则需要加上extra-android-m2repository，extra-android-support这两项。最后一项是设置虚拟机版本，如果本次构建不需要用到虚拟机则不需要加。

因为我们本次构建需要用到虚拟机来做Instrucment Test，所以需要打开虚拟机，可以在before_script中做这项工作

```
before_script:
    - echo no | android create avd --force -n test -t $ANDROID_TARGET --abi $ANDROID_ABI
    - emulator -avd test -no-skin -no-audio -no-window &
    - android-wait-for-emulator   # 该项会等待虚拟机启动成功在执行后面的命令
    - adb shell input keyevent 82 &
```

在script中用一些命令行指令来完成build工作。因为，我所用的项目只需要运行所有的Instrument单元测试用例并生成测试覆盖率报告，所以只需要加以下这句，这个Gradle任务是由Jacoco创建的，后面会详细介绍。注意--info最好加上，因为如果10分钟内没有输出，Travis—CI会认为build失败而结束build，--info让每完成一个测试方法都输出结果，因此能够防止这点。当然有可能单个测试方法运行时间超过10分钟的情况（Google Android 模拟器的蛋疼性能），所以单个测试方法应该竟可能小。

```
script:
    - ./gradlew :lib:createDebugAndroidTestCoverageReport --info --stacktrace
```

after_success，顾名思义，在成功完成构建后会执行。这里的任务是将测试覆盖率报告发送给Codecov，后面会详细介绍。

```
after_success:
    - bash <(curl -s https://codecov.io/bash)
```

## Jacoco

使用Jacoco生成测试覆盖率报告。只需要以下步骤：

1) 添加依赖

```
    androidTestCompile 'com.android.support.test:runner:0.4.1'
    // Set this dependency to use JUnit 4 rules
    androidTestCompile 'com.android.support.test:rules:0.4.1'
    // Set this dependency to build and run Espresso tests
    androidTestCompile 'com.android.support.test.espresso:espresso-core:2.2.1'
    // Espresso-contrib for DatePicker, RecyclerView, Drawer actions, Accessibility checks, CountingIdlingResource
    androidTestCompile 'com.android.support.test.espresso:espresso-contrib:2.2.1'
```
2) 在需要构建测试覆盖率报告的Module的gradle文件中设置。

```
debug{
        testCoverageEnabled true
    }
```

3) 运行 ./gradlew :module_name:createDebugAndroidTestCoverageReport 生成测试报告。测试报告地址：moudle_name/build/reports/coverage/debug/index.html。module_name为生成测试覆盖率的Moudle的名字。

## Codecov

Codecov不支持自己生成Android的测试覆盖率报告，它能做的是接收Jacoco生成的报告并进行可视化，也就是上面那个表示测试覆盖率的小图标。

集成Codecov只需要以下几个步骤。

1. 使用Github账号登录 https://codecov.io/， 并提供授权给该应用。
2. 在.travis.yml文件中添加命令将测试覆盖率报告上传给Codecov。

上面提到的

```
after_success:
    - bash <(curl -s https://codecov.io/bash)
```

就是将报告上传给Codecov。

## 总结

除了上面提到的，Travis还能做很多事情，比如自动打包发布等，留待读者自己探索。

## 参考

[http://jeroenmols.com/blog/2015/11/13/traviscoveralls/](http://jeroenmols.com/blog/2015/11/13/traviscoveralls/)

[https://docs.travis-ci.com](https://docs.travis-ci.com)

[https://segmentfault.com/a/1190000004415437](https://segmentfault.com/a/1190000004415437)

