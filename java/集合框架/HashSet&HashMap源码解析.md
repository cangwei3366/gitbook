---
title: Map - HashSet & HashMap源码解析
date: 2021-03-18 14:49:15
permalink: /pages/d37b6e/
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

HashMap实现了Map接口，允许key和value为null，除了未实现同步外，其他与hashTable类似；该容器并不保证有序。根据需要，例如在扩容时可能会对元素重新进行hash，元素的顺序也会再次打乱，因此不同时间迭代同一个hashmap元素的顺序可能不同。

根据对冲突的处理方式不同，哈希表有两种实现方式，一种开放地址方式(Open addressing)，另一种是冲突链表方式(Separate chaining with linked lists)。**Java HashMap采用的是冲突链表方式**。

![](../../.gitbook/assets/01_05_00.png)



有两个参数可以影响*HashMap*的性能: 初始容量(inital capacity)和负载系数(load factor)。初始容量指定了初始`table`的大小，负载系数用来指定自动扩容的临界值。当`entry`的数量超过`capacity*load_factor`时，容器将自动扩容并重新哈希。对于插入元素较多的场景，将初始容量设大可以减少重新哈希的次数。

将对象放入到*HashMap*或*HashSet*中时，有两个方法需要特别关心: `hashCode()`和`equals()`。**`hashCode()`方法决定了对象会被放到哪个`bucket`里，当多个对象的哈希值冲突时，`equals()`方法决定了这些对象是否是“同一个对象”**。所以，如果要将自定义的对象放入到`HashMap`或`HashSet`中，需要*@Override*`hashCode()`和`equals()`方法。

## HashMap实现

#### 底层结构

```java
	//初始容量16
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
    //负载因子
	static final float DEFAULT_LOAD_FACTOR = 0.75f;
    final float loadFactor;
    
    static final int UNTREEIFY_THRESHOLD = 6;
    //阈值 = loadFactor * capacity
    int threshold;
```

#### get()

```java
 //查找
    public V get(Object key) {
    	Node<K,V> e;
    	return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
    //得出key值对应hash值
    static final int hash(Object key) {
        int h;
        //上16与下16位异或 均匀hash
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
	
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) { //位运算求余 定位键值对所在桶的位置
            if (first.hash == hash && //判断首节点的key的hash值、键值是是否等于目标值
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {//否则向后遍历
                if (first instanceof TreeNode)//如果节点是treenode类型，则调用红黑树查找方法
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                //否则对链表进行查找
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

#### put()

```java

     public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    /**
     * Implements Map.put and related methods
     *
     * @param key的hash值
     * @param key
     * @param value
     * @param onlyIfAbsent if true, key存在的话不替换
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //初始化桶数组，初始化被延迟到第一次插入数据时
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //如果对应位置不存在键值，则创建一个节点并放到桶数组对应下标
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            //hash值、键值等于首节点，则替换
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //否则判断节点类型
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    //遍历到尾节点位置
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
						//除去头节点，如果链表节点个数达到7个，则开始红黑树转换
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //如果该键值在链表中存在，则中断
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                //onlyIfAbsent决定是否仅在为空的情况下替换
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        //扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            //当旧数组容量达到最大值时 不再扩容
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //按旧容量和阈值的两倍扩容
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            /*
             *初始化时 将threshold的值赋值给newcap
             *hashmap使用threshold变量暂时保存initcapacity的值
             */
            newCap = oldThr;
        else {               // 默认参数初始化
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        //newThr为0时重新计算阈值
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        //创建新的桶数组，同时初始化也是在这里实现
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            //如果旧的桶数组不为空，则将键值对映射到新的桶数组中
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    //如果仅有一个节点时
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        //如果后继节点是红黑树类型，则进行拆分
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        /*
                        * 拆分成两个链表 (e.hash & oldCap) == 0和(e.hash & oldCap) != 0
                        * 分别放在newTab[j + oldCap] 和newTab[j] 
                        */
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
  final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
            TreeNode<K,V> b = this;
            // Relink into lo and hi lists, preserving order
            TreeNode<K,V> loHead = null, loTail = null;
            TreeNode<K,V> hiHead = null, hiTail = null;
            int lc = 0, hc = 0;
            for (TreeNode<K,V> e = b, next; e != null; e = next) {
                //红黑树保留了链表的next引用，则可以按链表方式分组
                next = (TreeNode<K,V>)e.next;
                e.next = null;
                if ((e.hash & bit) == 0) {
                    if ((e.prev = loTail) == null)
                        loHead = e;
                    else
                        loTail.next = e;
                    loTail = e;
                    ++lc;
                }
                else {
                    if ((e.prev = hiTail) == null)
                        hiHead = e;
                    else
                        hiTail.next = e;
                    hiTail = e;
                    ++hc;
                }
            }
			
            if (loHead != null) {
                //如果lohead不为空，且列表长度小于6
                if (lc <= UNTREEIFY_THRESHOLD)
                    //则取消树化
                    tab[index] = loHead.untreeify(map);
                else {
                    tab[index] = loHead;
                    if (hiHead != null) 
                        //如果hihead为空，则树化
                        loHead.treeify(tab);
                }
            }
            if (hiHead != null) {
                if (hc <= UNTREEIFY_THRESHOLD)
                    tab[index + bit] = hiHead.untreeify(map);
                else {
                    tab[index + bit] = hiHead;
                    if (loHead != null)
                        hiHead.treeify(tab);
                }
            }
        }
}
```

