---
layout: post
title: 浅析ReactiveX的多播——实现安卓双击检测遇到的坑
date: 2016-3-31
categories: blog
author: beautifulSoup
tags: [Android, Development, RxJava]
description: 本文主要介绍如何在一个Observable订阅多个Subscriber。

---

## 背景
今天需要实现一个双击检测功能，以前的实现方式是自己记录上次点击时间与本次比对，如果小于门槛值，则发出双击事件。不过自从入了Rx的坑之后，凡事都喜欢用Rx的思想思考问题。于是上Github找找代码，还真找到一段，虽然是Kotlin的[一段错误的代码](https://gist.github.com/imton/ee74249fabff5ac95b16)，翻译成Java如下：（注：这段代码是有问题的，请不要看到这里就复制黏贴）


```java
	public void doubleClickDetect(View view){
        Observable<Void> observable = RxView.clicks(view);
        observable.buffer(observable.debounce(200, TimeUnit.MILLISECONDS))
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Action1<List<Void>>() {
                    @Override
                    public void call(List<Void> voids) {
                        //double click detected
                    }
                }, new Action1<Throwable>() {
                    @Override
                    public void call(Throwable throwable) {
                        Timber.e(throwable, "error");
                    }
                });
    }

```
我们先不讲这段代码的问题在哪，先来说说其中涉及到的几个操作符。
其中buffer()会将observable 发射的item缓存起来，直到它的参数Observable发射一个item时，它会将之前缓存的都作为一个list发射出去，这个还是比较好理解的，如果还是不懂，请参考 [buffer操作符文档](http://reactivex.io/documentation/operators/buffer.html)。

而debounce操作符会将在那些在参数时间间隔内跟着另一个item的items过滤掉。![](https://img.alicdn.com/imgextra/i3/754328530/TB2aHeWmXXXXXaCXXXXXXXXXXXX-754328530.png)
如图，1，5和6之后在一段时间内没有跟随其它item，所以被发射出来，而其它的都被过滤了。详见[debounce操作符文档](http://reactivex.io/documentation/operators/debounce.html)

所以上面代码的意思就是当有点击事件item产生的时候先缓存起来，当一段事件内没有新的事件产生的时候把之前缓存的事件作为一个列表发射出去，当发现有大于等于2的事件时，认为用户在一定时间内连续点了两次。所以这段代码在逻辑上是ok的。但是实际运行起来发现没反应，buffer后面没有item被发射。但是在buffer之前是有的，所以将问题定位到debounce没有item产生。所以问题在哪呢？我们需要明确一点，普通的Observable是不支持多播的，即使被多个Subscriber所订阅，也只会有一个Subscriber收到items。在buffer中，其实订阅了参数Observable，但是这个Observable在buffer之后又被订阅了一次，所以debounce就收不到item了。

## 修改
那么该如何修改以上代码，让它达到我们需要的效果呢。下面先来介绍几个相关的操作符。
### Publish
通过Publish操作符可以将一个普通的Observable转换为一个Connectable Observable。Connectable Observable 可以被多次订阅，被多个Subscriber共享Stream。但是和普通的Observable不同，它在被subscribe之后并不开始产生item，而需要在调用connect()之后才会产生item。

### Connect
在Publish中已经提到，用来让Connectable Observable开始产生item。

### Refcount
除了Connect，我们有另一种方式来让Connectable Observable 产生item，那就是Refcount，refCount会在第一个subscriber订阅之后自动connect，在最后一个subscriber unsubscribe之后自动disconnect。

### Share
Share 其实就是publish().refCount();

基于以上操作符，我们可以修正我们上面的代码了，稍微改一下就能达到我们预期的效果了，代码如下：

```java
	public void doubleClickDetect(View view){
        Observable<Void> observable = RxView.clicks(view).share();
        observable.buffer(observable.debounce(200, TimeUnit.MILLISECONDS))
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Action1<List<Void>>() {
                    @Override
                    public void call(List<Void> voids) {
                        //double click detected
                    }
                }, new Action1<Throwable>() {
                    @Override
                    public void call(Throwable throwable) {
                        Timber.e(throwable, "error");
                    }
                });
    }

```
上面的修改在原有代码基础上使用share操作符将原本的Observable变成了可共享流的Connectable Observable。

## 注意事项
在使用refCount或者share的时候需要注意一点，那就是在第一个subscriber订阅之后Connectable Observable就被connect产生item了。所以后面subscribe 的订阅者可能就收不到之前的一些item了。如果需要所有的subscriber都收到一样的item。还是先subscribe，最后再connect吧。

