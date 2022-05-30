---
title: Collection - Stack & Queue源码解析
date: 2021-03-18 14:48:25
permalink: /pages/986d92/
categories:
  - Java
  - Java进阶-集合框架
tags:
  - 
---

## JDK版本

```
JDK 1.8.0-144
```

## 概述

### Stack实现

#### 类关系

#### 底层结构

Stack继承了父类Vector结构，所以其底层存储为Object数组。

```
  protected Object[] elementData;
```

#### push

```java
 public E push(E item) {
        addElement(item);

        return item;
    }
 public synchronized void addElement(E obj) {
        modCount++;
        ensureCapacityHelper(elementCount + 1);
        elementData[elementCount++] = obj;
    }
```

#### 扩容

```java
   private void ensureCapacityHelper(int minCapacity) {
        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
  	private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                         capacityIncrement : oldCapacity);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

```

#### Queue实现

#### Deque实现

