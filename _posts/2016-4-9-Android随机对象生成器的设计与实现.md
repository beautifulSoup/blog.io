---
layout: post
title: Android随机对象生成器的设计与实现
date: 2016-4-8
categories: blog
author: beautifulSoup
tags: [Development,Android]
description: 本篇博客主要阐述如何实现一个随机对象生成器，并附上Github项目地址。
---
## 目标
当完成一个新的Feature的时候，需要对其进行测试。但是由于服务器还没有部署该功能，或者单元测试的限制，往往需要程序员自己去伪造一些数据。但是手工伪造数据往往效率不高并且没有代表性。因此希望能够实现一个对象生成器，生成对象并往里面填充随机值。

## 项目地址
[rog](https://github.com/campusappcn/rog)

## 设计要点
对象生成器的总体思路是清晰的，获取到类，使用反射获取到其所有的域，采用dfs(深度优先搜索)遍历类的域树，在遍历过程中设置随机值。

### 类型分类
针对Java类的特点，可以将其简单分为几种类型。
#### 基础类型
基础类型包括int、float、double、short、long、byte、char、boolean、String，需要针对这些基础类型提供默认的随机产生器。

#### Array
数组类型，默认随机产生器能够产生一个指定类的对象数组，并能够对数组成员赋随机对象。

#### Enum
枚举类型。默认构造器能够随机产生一个枚举值。

### Interface Or Abstract
对于接口和抽象类，我们需要首先知道它们的子类，否则无法产生其实例。在这里，有两种方案，方案一是扫描类路径下的所有类，找到该接口或抽象类的所有非抽象的子类，或者在编译期就生成继承树并记录下来。但是考虑到这样实现会比较复杂，所以在第一个版本，并没有按照这样的方案实现。如果读者对该方案有兴趣的，可以参考[reflections](https://github.com/ronmamo/reflections)。它是一个Java的开源项目，但是由于使用了Java7的一些特性，所以无法直接使用到Android项目中。接下来，我们说方案二，其实很简单，就是由使用者通过接口告诉rog某接口或者抽象类的非抽象子类有哪些，Rog会从中随机选择一个并产生它的实例返回。

### 其它类
除开上面提到的一些特殊的类，剩下的就是普通的一些类了，其它类的构造器ClassGenerator需要依赖于以上提到的构造器，将对应的一些的类对象的产生作业代理给以上产生器。

### UML图
![](https://img.alicdn.com/imgextra/i1/754328530/TB2113WmFXXXXb7XXXXXXXXXXXX_!!754328530.png)
上图是rog的整体UML图，从图中我们可以看到所有的Generator都继承自IGenerator接口，接口包含两个方法，generate()方法用来产生随机对象，getClassToGenerate()获取该Genreator所产生的对象类型。    

针对所有基本类型都实现了相对应的Generator，并提供了一些方法用于限制随机值的产生，比如设置最大值，设置不产生负数等。这些基本类型对象产生器通过BasicTypeGeneratorFactory进行管理。这是一个全局的单例。

上面的BasicTypeGeneratorFactory实现了ITypeGeneratorFactory接口。同样实现了该接口的还有TypeGeneratorFactory，该类用来缓存对象生成器，之前的Generator都会缓存到该Factory中，提升性能。该类依赖了BasicTypeGeneratorFactory，对于基本类型的产生器的获取会代理给BasicTypeGeneratorFactory。

在整个rog中，最重要的类就是ClassGenerator，该类可以产生所有类型的对象，不管是基本类型，还是抽象类。它依赖了以上提到的所有默认提供的Generator，将对应的一些的类对象的产生作业代理给以上产生器。

## 注意事项

### final的处理
对于有final修饰的域不再进行赋值。

### 产生层级限制
层级定义类引用的层数，比如Class1 为0层，且它有一个Class2的域，则该域的值对象的层级为1。如果不进行限制，则可能因为递归次数太多，而导致StackOverFlow或者无限循环无法结束程序。比如Class1有一个Class1类型的域。默认的最大层级为5，我们建议最大层级不要超过10。

### 域的缓存
由于反射效率低下，所以通过反射获取到某个类的域列表时，应该将其缓存起来。
