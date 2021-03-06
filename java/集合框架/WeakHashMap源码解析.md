---
title: Map - WeakHashMap源码解析
date: 2021-03-18 14:51:02
permalink: /pages/60884d/
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

WeakHashMap实现了Map接口，基于hash-map实现，在这种Map中，设置key的类型是WeakReference（弱引用）。如果对应的key被回收，则这个key指向的对象会被从Map容器中移除。

WeakHashMap跟普通的HashMap不同，WeakHashMap的行为一定程度上基于垃圾收集器的行为，因此一些Map数据结构对应的常识在WeakHashMap上会失效—size()方法的返回值会随着程序的运行变小，isEmpty()方法的返回值会从false变成true等等。

> WeakHashMap\*的这个特点特别适用于需要缓存的场景。 

## 实现

### 类依赖

![](../../.gitbook/assets/01_08_00.png)

### 底层结构

可以看到WeakHashMap的底层结构类似HashMap，但是增加了一个ReferenceQueue队列，	由上文的得知，WeakHashMap的key是弱键类型，即Entry的key为 WeakReference 类型，且Entry是键值对链表，具体介绍见下文。

当某弱键”不再被其它对象引用，并被GC回收时。在GC回收该“弱键”时，这个“弱键”也同时会被添加到ReferenceQueue(queue)队列中。  当下一次我们需要操作WeakHashMap时，会先同步table和queue。table中保存了全部的键值对，而queue中保存被GC回收的键值对；同步它们，就是删除table中被GC回收的键值对。 

```java
	 /**
     * The default initial capacity -- MUST be a power of two.
     */
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    /**
     * The maximum capacity, used if a higher value is implicitly specified
     * by either of the constructors with arguments.
     * MUST be a power of two <= 1<<30.
     */
    private static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * The load factor used when none specified in constructor.
     */
    private static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * The table, resized as necessary. Length MUST Always be a power of two.
     */
    Entry<K,V>[] table;

    /**
     * The number of key-value mappings contained in this weak hash map.
     */
    private int size;

    /**
     * The next size value at which to resize (capacity * load factor).
     */
    private int threshold;

    /**
     * The load factor for the hash table.
     */
    private final float loadFactor;

    /**
     * Reference queue for cleared WeakEntries
     */
    private final ReferenceQueue<Object> queue = new ReferenceQueue<>();

    /**
     * The number of times this WeakHashMap has been structurally modified.
     * Structural modifications are those that change the number of
     * mappings in the map or otherwise modify its internal structure
     * (e.g., rehash).  This field is used to make iterators on
     * Collection-views of the map fail-fast.
     *
     * @see ConcurrentModificationException
     */
    int modCount;




```

**Entry的结构** 

```java
 private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
        V value;
        final int hash;
        Entry<K,V> next;

        /**
         * Creates new entry.
         */
        Entry(Object key, V value,
              ReferenceQueue<Object> queue,
              int hash, Entry<K,V> next) {
            super(key, queue);
            this.value = value;
            this.hash  = hash;
            this.next  = next;
        }
        	....

```

**同步机制**

```java
  private void expungeStaleEntries() {
        for (Object x; (x = queue.poll()) != null; ) {
            synchronized (queue) {
                @SuppressWarnings("unchecked")
                    Entry<K,V> e = (Entry<K,V>) x;
                int i = indexFor(e.hash, table.length);

                Entry<K,V> prev = table[i];
                Entry<K,V> p = prev;
                while (p != null) {
                    Entry<K,V> next = p.next;
                    if (p == e) {
                        if (prev == e)
                            table[i] = next;
                        else
                            prev.next = next;
                        // Must not null out e.next;
                        // stale entries may be in use by a HashIterator
                        e.value = null; // Help GC
                        size--;
                        break;
                    }
                    prev = p;
                    p = next;
                }
            }
        }
    }
```



### 使用场景

**典型使用场景： tomcat两级缓存**

```java
package org.apache.tomcat.util.collections;

import java.util.Map;
import java.util.WeakHashMap;
import java.util.concurrent.ConcurrentHashMap;

public final class ConcurrentCache<K,V> {

    private final int size;

    private final Map<K,V> eden;

    private final Map<K,V> longterm;

    public ConcurrentCache(int size) {
        this.size = size;
        this.eden = new ConcurrentHashMap<>(size);
        this.longterm = new WeakHashMap<>(size);
    }

    public V get(K k) {
        V v = this.eden.get(k);
        if (v == null) {
            synchronized (longterm) {
                v = this.longterm.get(k);
            }
            if (v != null) {
                this.eden.put(k, v);
            }
        }
        return v;
    }

    public void put(K k, V v) {
        if (this.eden.size() >= size) {
            synchronized (longterm) {
                this.longterm.putAll(this.eden);
            }
            this.eden.clear();
        }
        this.eden.put(k, v);
    }
}
```

源码中有eden和longterm的两个map，对jvm堆区有所了解的话，可以猜测出tomcat在这里是使用ConcurrentHashMap和WeakHashMap做了分代的缓存。

在put方法里，在插入一个k-v时，先检查eden缓存的容量是不是超了。没有超就直接放入eden缓存，如果超了则锁定longterm将eden中所有的k-v都放入longterm。再将eden清空并插入k-v。
在get方法中，也是优先从eden中找对应的v，如果没有则进入longterm缓存中查找，找到后就加入eden缓存并返回。
经过这样的设计，相对常用的对象都能在eden缓存中找到，不常用（有可能被销毁的对象）的则进入longterm缓存。而longterm的key的实际对象没有其他引用指向它时，gc就会自动回收heap中该弱引用指向的实际对象，弱引用进入引用队列。longterm调用expungeStaleEntries()方法，遍历引用队列中的弱引用，并清除对应的Entry，不会造成内存空间的浪费。

**利用WeakHashMap实现内存缓存**

可以看出，WeakHashMap的这种特性比较适合实现类似本地、堆内缓存的存储机制——缓存的失效依赖于GC收集器的行为。假设一种应用场景：我们需要保存一批大的图片对象，其中values是图片的内容，key是图片的名字，这里我们需要选择一种合适的容器保存这些对象。

使用普通的HashMap并不是好的选择，这些大对象将会占用很多内存，并且还不会被GC回收，除非我们在对应的key废弃之前主动remove掉这些元素。WeakHashMap非常适合使用在这种场景下，下面的代码演示了具体的实现：

```java
WeakHashMap<UniqueImageName, BigImage> map = new WeakHashMap<>();
BigImage bigImage = new BigImage("image_id");
UniqueImageName imageName = new UniqueImageName("name_of_big_image"); //强引用

map.put(imageName, bigImage);
assertTrue(map.containsKey(imageName));

imageName = null; //map中的values对象成为弱引用对象
System.gc(); //主动触发一次GC

await().atMost(10, TimeUnit.SECONDS).until(map::isEmpty);
```

首先，创建一个WeakHashMap对象来存储BigImage实例，对应的key是UniqueImageName对象，保存到WeakHashMap里的时候，key是一个弱引用类型。

然后，我们将imageName设置为null，这样就没有其他强引用指向bigImage对象，按照WeakHashMap的规则，在下一次GC周期中会回收bigImage对象。

通过System.gc()主动触发一次GC过程，然后可以发现WeakHashMap成为空的了。
