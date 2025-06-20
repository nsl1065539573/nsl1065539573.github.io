+++
date = '2025-06-20T11:19:24+08:00'
draft = false
title = 'HashMap详解'
tags = ['java基础']
+++
---
## 一、HashMap底层原理
HashMap是java提供的一个高效的键值对结构，其底层采用数组加链表的形式来存储数据。
```java
  /**
     * The table, initialized on first use, and resized as
     * necessary. When allocated, length is always a power of two.
     * (We also tolerate length zero in some operations to allow
     * bootstrapping mechanics that are currently not needed.)
     */
    transient Node<K,V>[] table;
```
如上，底层采用数组存储元素，每个数组的索引处为一个链表，每个节点存储HashMap的key以及value。当需要put一个键值对时，hashMap会先根据key计算出hash值，根据hash值计算出键应该被放在数组的哪个位置，之所以使用链表，是因为hashMap采用拉链法的方式来解决hash冲突，将新的节点插入到链表尾部，在检索时判断key是否存在以及是否equals来决定键是否在hashMap中。

### 1. put操作
```java
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```
put操作如上，总共分为以下几个步骤：
1. 如果表不存在则创建新表
2. 如果对应位置为空，则直接新建节点，并且把当前节点作为链表头结点放入对应索引位置处
3. 如果是红黑树，则使用红黑树的新增节点
4. 遍历链表，如果key已经存在，则直接取消循环(后面更新这个节点的值)
5. 如果key不存在，则在链表尾部新增节点
6. 如果到达了转红黑树的阈值则转红黑树
7. 最后判断如果超过了阈值则触发扩容

### 扩容操作
```java
  final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
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
```
扩容操作如上，在map内元素到达阈值之后，会触发扩容，声明一个新的数组，之后更新新的扩容阈值，将元素重新hash计算位置并赋值到新的数组中。

### get操作
```java
  public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(key)) == null ? null : e.value;
    }

    /**
     * Implements Map.get and related methods.
     *
     * @param key the key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n, hash; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & (hash = hash(key))]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
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
get操作比较简单，根据key获取到数组对应位置，如果该位置为空，则返回空，如果不为空，则遍历树或者链表，当key比较相等时返回。

---
## 二、线程安全性
如上，可以发现hashMap的底层代码中是没有加锁的操作的，所以在多线程情况下操作同一个HashMap实例会有并发问题。主要有以下几个问题：
### 1. 数据覆盖
```java
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
```
当有两个线程同时竞争时，同时进入了此if判断，就会导致丢掉某一个线程的put操作。
1. A线程put，发现位置为空，进入if，A线程挂起
2. B线程put，发现位置为空，进入if，对`tab[i]`进行赋值
3. A线程获取CPU时间片，对`tab[i]`进行赋值
上述流程中，B线程的put操作丢失
### 2. 计数器错误
```java
if (++size > threshold)
    resize();
```
如上，++size在多线程情况下会导致少加等操作，影响扩容时机
### 3. 丢失数据
在扩容操作中，并没有对操作进行加锁，会导致数据丢失问题
假设A，B两个线程同时触发扩容
1. A创建新数组并将 table 指向它，但尚未完成迁移
2. B 开始迁移元素，访问 table 时看到的是 A 创建的新数组
3. B 迁移节点 A 到新数组位置 i
4. A 也迁移节点B到新数组位置 i，覆盖 B 的操作
结果：节点 A 丢失，无法被访问
