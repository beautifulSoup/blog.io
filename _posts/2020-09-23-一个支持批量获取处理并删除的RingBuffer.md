---
layout: post
title: 一个支持批量获取处理并删除的RingBuffer
date: 2020-09-23
categories: blog
tags: [Java,RingBuffer,CircularBuffer,环形队列]
description: 
---
# 一个支持批量获取处理并删除的RingBuffer

在工作中遇到一个应用场景，有多个Producer生产一些任务，然后由一个Consumer批量获取并处理，如果批量处理失败了需要回滚，下次获取重新获取到上次处理失败的数据并重新尝试处理。如果存储任务的容器满了，则需要阻塞生产者线程。在遇到这个场景时，第一时间就想到了RingBuffer，但是很多Java扩展包里面的RingBuffer实现并不支持批量获取，也不支持二段ack确认删除，只能一次获取一个并从RingBuffer中删除。因此对org.apache.commons.collections.buffer.BoundedFifoBuffer类进行了一定的改造实现了该需求。

![RingBuffer应用场景](http://img.tangokk.com/2020-09-23-083448.png)

代码如下，注意该环形队列支持多生产者同时生产（用锁实现），但是消费者只能有一个。另外当环形队列满时，会阻塞生产线程。

```java
package com.hcrm.mall.goods.syncer.buffer;


import java.util.Arrays;
import java.util.concurrent.Semaphore;
import java.util.concurrent.locks.ReentrantLock;
import org.apache.commons.lang.ArrayUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * 允许多线程写
 * 只允许单线程->读->处理->移除
 */
public class CircularFifoBuffer {

    private Logger logger = LoggerFactory.getLogger(CircularFifoBuffer.class.getName());


    private transient Object[] elements;

    private transient int start = 0;
    private transient int end = 0;

    private transient boolean full = false;

    private final int maxElements;

    private ReentrantLock addLock;

    private Semaphore semaphore;

    public CircularFifoBuffer(int size) {
        if (size <= 0) {
            throw new IllegalArgumentException("The size must be greater than 0");
        }
        elements = new Object[size];
        maxElements = elements.length;
        addLock = new ReentrantLock();
        semaphore = new Semaphore(size);
    }


    public int size() {
        int size = 0;

        if (end < start) {
            size = maxElements - start + end;
        } else if (end == start) {
            size = (full ? maxElements : 0);
        } else {
            size = end - start;
        }

        return size;
    }

    public boolean isEmpty() {
        return size() == 0;
    }

    public boolean isFull() {
        return size() == maxElements;
    }

    public int maxSize() {
        return maxElements;
    }

    public void clear() {
        full = false;
        start = 0;
        end = 0;
        Arrays.fill(elements, null);
    }

    public boolean add(Object element) {
        if (null == element) {
            throw new NullPointerException("Attempted to add null object to buffer");
        }

        addLock.lock();
        try {
            semaphore.acquire();
        } catch (Exception e) {
            logger.error("RingBuffer", "线程退出，添加失败");
            return false;
        }

        elements[end++] = element;


        if (end >= maxElements) {
            end = 0;
        }

        if (end == start) {
            full = true;
        }

        addLock.unlock();

        return true;

    }

    public Object get() {
        if (isEmpty()) {
            return null;
        }

        return elements[start];
    }


    public Object remove() {
        if (isEmpty()) {
            return null;
        }

        Object element = elements[start];
        if(null != element) {
            elements[start++] = null;
            if (start >= maxElements) {
                start = 0;
            }
            full = false;
            semaphore.release();
        }
        return element;
    }


    /**
     * @param size the max size of elements will return
     */
    public Object[] get(int size) {
        int queueSize = size();
        if (queueSize == 0) { //empty
            return new Object[0];
        }
        int realFetchSize =  queueSize >= size ? size : queueSize;
        if (end > start) {
            return Arrays.copyOfRange(elements, start, start + realFetchSize);
        } else {
            if (maxElements - start >= realFetchSize) {
                return Arrays.copyOfRange(elements, start, start + realFetchSize);
            } else {
                return ArrayUtils.addAll(
                    Arrays.copyOfRange(elements, start, maxElements),
                    Arrays.copyOfRange(elements, 0, realFetchSize - (maxElements - start))
                );
            }
        }
    }


    public Object[] getAll() {
        return get(size());
    }



    public Object[] remove(int size) {
        if(isEmpty()) {
            return new Object[0];
        }
        int queueSize = size();
        int realFetchSize = queueSize >= size ? size : queueSize;
        Object [] retArr = new Object[realFetchSize];
        for(int i=0;i<realFetchSize;i++) {
            retArr[i] = remove();
        }

        return retArr;
    }



}

```



