---
layout: post
title: Android自定义Notification并没有那么简单
date: 2016-4-8
categories: blog
author: beautifulSoup
tags: [Development,Android]
description: 本文主要介绍自定义Notification中需要注意的一些地方。
---

# 背景

最近需要实现一个自定义Notification的功能。网上找了找代码，解决方案就是通过RemoteViews来实现。但是在实现过程中遇到不少问题，网上也没有很好的文章描述这些问题，所以在这里做个总结，希望大家能少走点弯路。

# 实现

## RemoteViews 自定义View
这是最基础的知识点，虽然做过自定义通知的应该都清楚，但我觉得还是有必要带一下。它主要被用于AppWidget和Notification，它描述一个在其它进程中显示的View。以下是例子代码。从中我们可以看到RemoteViews提供了一些方法来改变它的子View的值，如设置TextView的文字等。

```java
RemoteViews remoteViews = new RemoteViews(context.getPackageName(), R.layout.view_notification_type_0);
        remoteViews.setTextViewText(R.id.title_tv, title);
        remoteViews.setTextViewText(R.id.content_tv, content);
        remoteViews.setTextViewText(R.id.time_tv, getTime());
        remoteViews.setImageViewResource(R.id.icon_iv, R.drawable.logo);
        remoteViews.setInt(R.id.close_iv, "setColorFilter", getIconColor());
        Intent intent = new Intent(context, MainActivity.class);
        intent.putExtra(NOTICE_ID_KEY, NOTICE_ID_TYPE_0);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_CLEAR_TASK);
        int requestCode = (int) SystemClock.uptimeMillis();
        PendingIntent pendingIntent = PendingIntent.getActivity(context, requestCode, intent, PendingIntent.FLAG_UPDATE_CURRENT);
        remoteViews.setOnClickPendingIntent(R.id.notice_view_type_0, pendingIntent);
        int requestCode1 = (int) SystemClock.uptimeMillis();
        Intent intent1 = new Intent(ACTION_CLOSE_NOTICE);
        intent1.putExtra(NOTICE_ID_KEY, NOTICE_ID_TYPE_0);
        PendingIntent pendingIntent1 = PendingIntent.getBroadcast(context, requestCode1, intent1, PendingIntent.FLAG_UPDATE_CURRENT);
        remoteViews.setOnClickPendingIntent(R.id.close_iv, pendingIntent1);
```
这里有几点需要注意的。

### setInt

这个方法被用来调用子View中需要一个Int型参数的方法。如下面这句代码，调用了id为close_iv的setColorFilter方法，参数为getIconColor()的返回值。

```java
remoteViews.setInt(R.id.close_iv, "setColorFilter", getIconColor());
```

### 设置不同区域的点击PendingIntent

默认的Notification只能通过setContentIntent设置整体的点击事件。不过通过RemoteViews我们可以设置不同地方不同的点击事件，当然这里的事件指的是PendingIntent。如下，设置了点击R.id.notice_view_type_0打开一个Activity，而点击R.id.close_iv会发出一个广播，可以通过这个广播的广播接收器来做一些事情，如这里是关闭当前的Notification。另外还可以打开一个Service。

```java
PendingIntent.getActivity(context, requestCode, intent, PendingIntent.FLAG_UPDATE_CURRENT);
        remoteViews.setOnClickPendingIntent(R.id.notice_view_type_0, pendingIntent);
        int requestCode1 = (int) SystemClock.uptimeMillis();
        Intent intent1 = new Intent(ACTION_CLOSE_NOTICE);
        intent1.putExtra(NOTICE_ID_KEY, NOTICE_ID_TYPE_0);
        PendingIntent pendingIntent1 = PendingIntent.getBroadcast(context, requestCode1, intent1, PendingIntent.FLAG_UPDATE_CURRENT);
        remoteViews.setOnClickPendingIntent(R.id.close_iv, pendingIntent1);
```

# 设置通知的自定义View
以上我们得到了自定义的RemoteViews。通过下面这段代码就能生成自定义View的Notification，注意这里使用了setContent()方法。这是网上自定义Notification都会使用的方法。

```java
Notification notification = new NotificationCompat.Builder(context).setContent(remoteViews).build();
```
但是它会有一个问题。

通过setContent()方法获得的Notification是定高的。如果View的高度比默认高度要大的话，就有一部分显示不出来。如下图
![](https://img.alicdn.com/imgextra/i4/754328530/TB2Wiq.npXXXXXDXXXXXXXXXXXX_!!754328530.png)

默认情况下通知高度为64dp，当然Rom不同可能会有些区别。一般文字在小于两行的情况下都是可以显示。

那么如何做到wrap_content。需要使用一些黑科技。如下：

```java
NotificationCompat.Builder builder = new NotificationCompat.Builder(context);
if(android.os.Build.VERSION.SDK_INT >= 16) {
            notification = builder.build();
            notification.bigContentView = remoteViews;
}
notification.contentView = remoteViews;
```
为了理解以上代码，我们需要明确一个我们很容易忽略的问题，那就是通知是可以展开和收起的。请看以下两张图片。同样是网易云音乐的通知，图一比图二要大一些。其实图一展示的是网易云音乐通知的展开状态，使用两个手指上滑就可以缩起，也就是图二。
![](https://img.alicdn.com/imgextra/i2/754328530/TB2NY19npXXXXX_XXXXXXXXXXXX_!!754328530.png)
![](https://img.alicdn.com/imgextra/i1/754328530/TB2afuqnpXXXXXdXFXXXXXXXXXX_!!754328530.png)

在上面的代码中我们分别设置了bigContentView 这是展开的自定义视图，而contentView则是收起时的视图。

注意bigContentView是在sdk16时引入的，所以需要判断一下。如果小于sdk16则只能定高了。

注意bigContentView 的最大高度是100dp

注意bigContentView和contentView的设置不能调转顺序，亲测这样会让contentView不显示。

另外需要注意某些Rom可能不支持展开收起通知，在设置了BigContentView之后就只显示展开的视图，而默认情况下只展示收起视图。如魅族的FlyMe，其它Rom并没有测试，如果读者知道可以分享一下。

## 背景色适配

不同Rom的通知背景色是不同的，所以在UI上需要注意。
主要分为两种情况。

- 背景色为有透明度的黑色，如MiUi、FlyMe。
- 背景色为白色，如原生的5.0之后的Rom、华为部分Rom。

主要有两种方案。

### 固定背景色

也就是设置一个固定的背景色，文字和icon颜色都可以固定。如下图。
![](https://img.alicdn.com/imgextra/i2/754328530/TB2XoYanpXXXXXvXXXXXXXXXXXX_!!754328530.png)
这有一个缺点，我们在图中也看到了，那就是某些Rom的Notification会有一个左右的padding，如果固定背景色就会很难看。所以这种方法虽然简答，但是不建议使用。

### 透明背景色

另一种方法就是让北京透明。那么文字和icon的颜色怎么办呢？很简单，跟随系统的Notification中文字的颜色。如下设置了TextView的style为默认通知中info的样式。其它相关Style包括TextAppearance.StatusBar.EventContent.Line2、TextAppearance.StatusBar.EventContent.Info等。

```
        <TextView
            android:id="@+id/content_tv"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textAppearance="@style/TextAppearance.StatusBar.EventContent.Info"
            tools:text="41个同校小伙伴参与讨论"
            android:layout_marginTop="4dp"
            android:singleLine="true"/>
```

需要注意的一点是Android5.0之后使用了不同的Style名表示通知样式。
我们需要创建一个layout-v21文件夹，并新建一个在5.0之后使用的自定义通知样式。如下同样是设置TextView的style为Info的样式，但我们使用的是@android:style/TextAppearance.Material.Notification.Info。

```xml
<TextView
            android:id="@+id/content_tv"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textSize="9sp"
            android:textAppearance="@android:style/TextAppearance.Material.Notification.Info"
            tools:text="41个同校小伙伴参与讨论"
            android:layout_marginTop="4dp"
            android:singleLine="true"/>
```

另外如果自定义view中有Icon，那么Icon的颜色也需要适应背景，因为主要的背景颜色为透明黑色和白色，所以取灰色，如#999999，两种不同情况下的文字内容颜色都为该值，因此在两种背景上都能很好地显示。

或者根据不同的背景色设置不同的颜色，通过上面提到的setInt方法。ImageView的setColorFilter方法可以设置图案颜色为某种纯色。但是目前我还没有找到很好的方法获取默认通知的背景色，如果读者找到了望告知。

```java
remoteViews.setInt(R.id.close_iv, "setColorFilter", getIconColor());
```

# 最终效果

![](https://img.alicdn.com/imgextra/i4/754328530/TB2uf1pnpXXXXX6XFXXXXXXXXXX_!!754328530.png)
![](https://img.alicdn.com/imgextra/i3/754328530/TB2rdvanpXXXXXQXXXXXXXXXXXX_!!754328530.png)

# 总结

以上即为我在自定义Notification中遇到的一些问题以及解决方案。目前还有两点有待进一步完善。

- 获取默认通知背景色，或者使图标颜色与背景色适配的方案。
- 不支持Notification展开收起的Rom，目前知道的仅有FlyMe。

# 示例代码地址

[https://github.com/beautifulSoup/CNotification](https://github.com/beautifulSoup/CNotification)
