---
layout: post
title: 在Android项目中使用Lombok
date: 2016-12-21
categories: blog
tags: [Development,Android,lombok]
description: 本篇博客主要介绍在Android项目中使用lombok插件
---
<br/>
<br/>

## 前言
之前写了一下后台代码，发现后台项目中使用了一个很好用的插件——Lombok。它帮助程序员避免写一些setter、getter、toString等机械化的代码，减少了程序员的机械劳动。既然是Java项目，那么在Android中应该也是能用的，于是在Android项目中也尝试了一下。

## 依赖
如下是Gradle文件配置。因为Lombok的原理是根据注解生成代码，所以需要用到apt。
在Project的build.gradle文件中添加对apt的依赖

```
buildscript {
    repositories {
        jcenter()
        mavenCentral()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.1.2'
        //添加apt依赖
        classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
    }
}

```

在app的build.gradle文件中修改

```
//应用apt插件
apply plugin: 'com.neenbedankt.android-apt'
...

dependencies {
	    compile 'org.projectlombok:lombok:1.16.8'  //添加lombok依赖
	    ...
}
```

## 代码
lombok使用Annotation来申明某个类需要添加getter，setter等，下面是使用lombok和不使用lombok的对比。

```java
@Setter
@Getter
@ToString
public class XXX implements Entity {

    String id;
	
}
```

```java
public class XXX implements Entity {

    String id;
    
    public String getId(){
    	return this.id;
    }
    
    public void setId(String id){
    	this.id = id;
    }
}
```

可以看到我们不再需要手工去写Getter和Setter了。

## AS插件
添加了依赖之后，虽然编译时是正确的。但是因为Android Studio语法识别器不认识@Getter和@Setter注解，所以需要添加Lombok插件。
在设置页面 -> plugins -> browser repository -> 搜索lombok -> install 
成功安装之后，再写比如XXX.getId()方法时AS就不会报错了。

