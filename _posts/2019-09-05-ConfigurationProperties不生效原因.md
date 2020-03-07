---
layout: post
title: ConfigurationProperties不生效的一种原因
date: 2019-09-05
categories: blog
tags: [spring]
description: 本文主要介绍了一种ConfigurationProperties不生效的一种原因。

---

## ConfigurationProperties不生效的一种原因

@ConfigurationProperties 作为Spring中的一个注解主要用于读取配置文件的信息，并自动封装成实体类，这样子，我们在代码里面使用就轻松方便多了。

如

```java
@Component
@ConfigurationProperties(prefix="connection")
public class ConnectionSettings {

    private String username;
    private String remoteAddress;
    private String password ;

    public String getUsername() {
        return username;
    }
    public void setUsername(String username) {
        this.username = username;
    }
    public String getRemoteAddress() {
        return remoteAddress;
    }
    public void setRemoteAddress(String remoteAddress) {
        this.remoteAddress = remoteAddress;
    }
    public String getPassword() {
        return password;
    }
    public void setPassword(String password) {
        this.password = password;
    }

}

```

配置文件里面可以写成

```yaml
connection:
	username: tanglikang
	remoteAddress: www.ybdoctor.com
	password: 123456
```



但是在某些情况下配置信息不会被成功加载。实体类中的成员变量值为null。



可能有几个原因。

### setter没有设置

```java
@Component
@ConfigurationProperties(prefix="connection")
public class ConnectionSettings {

    private String username;
    private String remoteAddress;
    private String password ;

    public void setUsername(String username) {
        this.username = username;
    }
    public void setRemoteAddress(String remoteAddress) {
        this.remoteAddress = remoteAddress;
    }
    public void setPassword(String password) {
        this.password = password;
    }

}


```



