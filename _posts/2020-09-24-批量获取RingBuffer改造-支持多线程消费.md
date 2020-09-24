---
layout: post
title: 批量获取RingBuffer改造-支持多线程消费
date: 2020-09-23
categories: blog
tags: [Java,RingBuffer,CircularBuffer,环形队列]
description: 
---
# 批量获取RingBuffer改造-支持多线程消费

在上一篇博文 [一个支持批量获取处理并删除的RingBuffer](http://www.tangokk.com/blog/2020/09/23/%E4%B8%80%E4%B8%AA%E6%94%AF%E6%8C%81%E6%89%B9%E9%87%8F%E8%8E%B7%E5%8F%96%E5%A4%84%E7%90%86%E5%B9%B6%E5%88%A0%E9%99%A4%E7%9A%84RingBuffer/) 中我们介绍了使用RingBuffer实现的支持多线程写，单线程获取，处理之后ack的缓存队列。如下图

![](http://img.tangokk.com/2020-09-23-083448.png)

那我们是否能够继续优化，使其支持多线程读呢。这就让我们想到了分块的思想。通过把一个RingBuffer分块，分成几个Segment，每个消费者消费一个Segment，生产者在生产时根据生产对象的hashcode取余数写入不同的segment中。代码如下，由于已经有上一篇的RingBuffer的基础。因此我们直接依赖了CircularFifoBuffer。

```java
package com.tangokk.commons.buffer;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * 允许多线程写
 * 允许多线程->读->处理->移除，单个消费者只能消费一个segment
 */
public class SegmentsCircularFifoBuffer {

    private Logger logger = LoggerFactory.getLogger(SegmentsCircularFifoBuffer.class.getName());


    int segmentCount;

    CircularFifoBuffer [] segmentBuffers;

    public SegmentsCircularFifoBuffer(int size, int segmentCount) {
        if (size <= 0) {
            throw new IllegalArgumentException("The size must be greater than 0");
        }
        this.segmentCount = segmentCount;
        segmentBuffers = buildSegments(size, segmentCount);
    }



    private CircularFifoBuffer [] buildSegments(int size, int segmentCount) {
        int segmentSize = (int)Math.ceil(size / segmentCount);
        CircularFifoBuffer [] segments = new CircularFifoBuffer[segmentCount];
        for(int i = 0;i<segmentCount;i++) {
            segments[i] = new CircularFifoBuffer(segmentSize);
        }
        return segments;
    }


    public int size(int segmentId) {
        return segmentBuffers[segmentId].size();
    }

    public boolean isEmpty(int segmentId) {
        return segmentBuffers[segmentId].isEmpty();
    }

    public boolean isFull(int segmentId) {
        return segmentBuffers[segmentId].isFull();
    }

    public int maxSize(int segmentId) {
        return segmentBuffers[segmentId].maxSize();
    }

    public void clear(int segmentId) {
        segmentBuffers[segmentId].clear();
    }

    public boolean add(Object element) {
        if (null == element) {
            throw new NullPointerException("Attempted to add null object to buffer");
        }
        int segmentId = element.hashCode() % segmentCount;
        segmentBuffers[segmentId].add(element);
        return true;

    }

    /**
     * @param size the max size of elements will return
     */
    public Object[] get(int segmentId, int size) {
        return segmentBuffers[segmentId].get(size);
    }


    public Object[] getAll(int segmentId) {
        return segmentBuffers[segmentId].getAll();
    }



    public Object[] remove(int segmentId, int size) {
        return segmentBuffers[segmentId].remove(size);
    }
}
```



源码请看： [https://github.com/beautifulSoup/commons/blob/master/src/main/java/com/tangokk/commons/buffer/SegmentsCircularFifoBuffer.java](https://github.com/beautifulSoup/commons/blob/master/src/main/java/com/tangokk/commons/buffer/SegmentsCircularFifoBuffer.java)