---
layout: post
title: 每天学点ShellScript——16进制颜色值转换器
date: 2016-8-17
categories: blog
author: beautifulSoup
tags: [Development,Android]
description: 本文主要完成了一个16进制颜色值转换器 
---


# 每天学点ShellScript——16进制颜色值转换器

## 源码

写了个shell小脚本用来把%为单位的alpha值和RGB值转换为16进制数。

输入有三种情况：

- 一个参数： 认为是alpha值，转换为16进制的alpha值。
- 三个参数：认为是RGB 值，转换为6个字符表示16进制的颜色值。
- 四个参数：认为是alpha跟上RGB值，转换为8个字符表示alpha和颜色值的16进制数。

```bash
#! /bin/bash

toHex(){
    v=`echo "obase=16;${1}" | bc`
    l=`echo ${#v}`
    if [ $l -lt 2 ]
    then
        v="0${v}"
    fi
    echo $v
}

toAlpha(){
    echo "obase=16;$(($1*256/100))" | bc
}

toColorHex(){
    v1=`toHex $1`
    v2=`toHex $2`
    v3=`toHex $3`
    echo "${v1}${v2}${v3}"
}


if [ $# -eq 3 ]
then
    color=`toColorHex $1 $2 $3`
    echo "#$color"
elif [ $# -eq 4 ]
then
    alpha=`toAlpha $1`
    color=`toColorHex $2 $3 $4`
    echo "#${alpha}${color}"
elif [ $# -eq 1 ]
then
    echo `toAlpha $1`
else
    echo "You should input four or three numbers"
fi

```

## 主要知识点   
### 参数获取
从脚本或者函数内部都可以通过 1、2、3... 变量名获取传入的参数。${1}即为传入的第一个参数。而${#}获取参数个数。 

### 关系运算符
| 符号 | 含义      |
|-----|-----      |
| -eq | equal     |
| -ne | not equal |
| -lt | less than |
| -gt | great than|
| -ge | great than or equal |
| -le | less than or equal |

### 命令执行
如果需要将命令执行的结果赋值给变量，则需要将这条命令语句用``扩起，或者使用$(),不过$()在某些系统环境下可能无法使用。

### 数学计算
数学计算语句需要用$(())括起。

### bc
bc是一种科学语句，也是shell种一个命令，不过大多数的使用场景是用来做数值转换。这里就是转为16进制。

### [
shell 中的if 后面的[ 是一个可执行程序，所以需要在其后加空格，] 同理。

### 变量和等号之间不能有空格

### 获取字符串长度
使用${#string} 
