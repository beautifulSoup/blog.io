---
layout: post
title: Android开发安全手册
date: 2017-3-10
categories: blog
tags: [Development,Android]
description: 本篇博客主要给出了一些Android开发中的安全建议。
---


## 常规安全防御手段

### 混淆

混淆是Android基本安全手段，虽然目前有很多工具能够反混淆，但是对于反编译调试代码还是有较大作用的。

### 加固

目前有很多第三方加固服务可以使用。如爱加密、360加密、阿里聚安全等。可以选择一个使用。但是也不能认为使用了加固就万事大吉了，因为还是有被脱壳的风险的。

## 安全风险规避

### 数据接口泄露风险

#### 风险说明

数据接口被劫持，中间人攻击等。

#### 风险规避

1. 使用HTTPS协议。
2. HTTPS证书双向校验，客户端校验服务端证书、域名等是否一致，服务端校验客户端证书是否正确(签名CA是否合法、证书是否是自签名、主机域名是否匹配、证书是否过期等)。


### 本地存储数据泄露风险

#### 风险说明

Android提供四种Android本地存储方式，分别为Shared Preferences、SQLite Databases、Internal Storage、External Storage。对于一些不当的存储方式，可能会造成数据泄露。

### 风险规避

1. 对于SP、Database和Internal Storage可以设置打开方式，原则上都使用MODE_PRIVATE。
2. 谨慎使用sharedUserId属性。拥有相同sharedUserId 和签名的应用可以共享本地数据。所以拥有sharedUserId属性的应用一定要使用安全的证书签名。
3. 谨慎使用External Storage，因为它能够被所有应用读取。
4. 对于敏感信息，需要加密之后进行存储，如密码等。


### 硬编码密钥泄露风险

#### 风险说明

将AES加密密钥、支付宝SDK密钥等硬编码在代码中容易被反编译获得。

#### 风险规避

对密钥进行加密，在使用的时候解密获得。推荐使用阿里聚安全的sdk加密。

### 日志泄露风险

#### 风险说明

打印在Log中的日志容易被调试获取一些信息，所以应该避免在Release版本中输出敏感日志。

#### 风险规避

推荐使用Timber等日志框架。设置在Release版本中不输出日志。

### WebView远程代码执行漏洞

#### 风险说明

api16之前，可以通过js反射获取Java代码运行环境，然后进行远程代码执行。

#### 风险防范

1. 如果没有必要，设置WebView不支持Js接口调用。

   ```java
   webView.getSettings().setJavascriptEnabled(false);
   ```

2. 移除系统隐藏JS调用接口

   ```java
   webView.removeJavascriptInterface("searchBoxJavaBridge_");
   webView.removeJavascriptInterface("accessibilityTraversal");
   webView.removeJavascriptInterface("accessibility");
   ```

3. 如果WebView不需要支持打开第三方网页，则可以过滤第三方网页，只允许打开自己的网页。可以使用HTTPS证书校验、白名单等方式进行过滤。

4. 使用不支持Js接口调用的WebView打开第三方网页。

### WebView 明文存储密码漏洞

#### 风险说明

在api17之前，WebView会把用户统一保存的密码明文存储在手机上。因此可能会泄露用户密码。

#### 风险规避

```java
WebView.setSavaPassword(false);
```

### 本地拒绝服务风险

如果Activity的exported属性为true，则它能够被外部打开。这时候如果外部传给该Activity的数据异常，则可能会使应用闪退。

如某个被攻击的Activity代码

```java
MyClass object = (MyClass)getIntent().getSerializableExtra("my_class");
```

则如果攻击者代码为，会造成被攻击者的闪退。

```java
Intent localIntent = new Intent();
localIntent.setComponent(new ComponentName(packageName, className));
localIntent.putExtra("my_class", new MalformedObject());
startActivity(localIntent);
```

#### 风险规避

1. 如非必要，不要设置Activity的exported属性为true。
2. 在获取Intent数据时检测过滤并catch异常。

### 弱加密风险

#### 风险说明

使用一些较容易破解的加密算法。

#### 风险规避

1. 使用对称加密算法时避免使用DES算法
2. 使用RSA算法加密时不使用NoPadding
3. IvParameterSpec初始化时，不使用常量vector
4. 在选择加密模式时避免使用ECB模式
5. 使用RSA加密时，建议密钥长度大于1024bit









