+++
date = '2025-06-20T16:50:56+08:00'
draft = false
title = 'ConcurrentHashMap详解'
tags = ['java基础']
+++

---
## 一、什么是ConcurrentHashMap
`ConcurrentHashMap`是java提供的一个线程安全的HashMap，上一篇文章阐述了HashMap线程不安全的原因以及在多线程环境下会产生的问题，本文将介绍线程安全的HashMap。

---
## 二、实现原理
1. `ConcurrentHashMap`通过槽的形式减小锁的粒度，数组的每个索引对应一个槽，每次对当前槽的头结点加锁
2. 采用CAS的形式进行扩容等操作，减少锁竞争
3. 字段采用`volatile`关键字修饰，使字段可以在线程中共享

### put操作
```java
    transient volatile Node<K,V>[] table;

    public V put(K key, V value) {
        return putVal(key, value, false);
    }

    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh; K fk; V fv;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else if (onlyIfAbsent // check first node without acquiring lock
                     && fh == hash
                     && ((fk = f.key) == key || (fk != null && key.equals(fk)))
                     && (fv = f.val) != null)
                return fv;
            else {
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key, value);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                        else if (f instanceof ReservationNode)
                            throw new IllegalStateException("Recursive update");
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```
如上，通过使用`volatile`关键字修饰数组来保证可见性，在数组为空或者对应索引处为空时采用CAS的形式进行设置值，否则对当前头结点加锁，进行put操作。最后增加哈希表的size，如果要扩容则触发扩容。

值得一提的是，在put操作时，如果发现当前槽已经被迁移，那么`ConcurrentHashMap`会帮助一起迁移数组，之后再进行put操作。

整个流程抽象为如下步骤：
1. 表是否为空，如果为空使用CAS来创建数组
2. 计算得到的应该插入的位置是否为空，如果为空，采用CAS设置对应位置的节点
3. 当前槽是否已被迁移，如果已经被迁移，则帮助触发迁移
4. 对当前槽的头结点加锁，如果是树节点则更新树节点，如果是链表则更新链表
5. 如果需要转树节点则转树节点
6. 增加size

### 扩容操作
#### 1. 扩容触发条件  
1. 当 `ConcurrentHashMap` 中的元素数量（`size`）超过扩容阈值（`sizeCtl`）时触发。  
2. 阈值计算方式：初始容量的 **3/4**（例如初始容量为 16 时，阈值为 12）。  


#### 2. 扩容准备阶段  
1. **生成扩容标记**  
   - 通过 `resizeStamp(n)` 生成 16 位扩容标记（`n` 为原数组长度），用于标识本次扩容。  
   ```java
   int rs = resizeStamp(n); // 标记格式：高16位为扩容标识，低16位为0
   ```  

2. **更新 `sizeCtl` 状态**  
   - 将 `sizeCtl` 设为负数，低 16 位表示参与扩容的线程数（初始为 `2`，即 1 个线程参与）。  
   ```java
   // 格式：[扩容标记(高16位)][线程数+1(低16位)]
   U.compareAndSetInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2);
   ```  

3. **创建新数组**  
   - 新数组长度为原数组的 **2 倍**（例如原长度 16 → 新长度 32）。  


#### 3. 多线程迁移阶段（核心流程）  
1. **任务分配机制**  
   - 每个线程负责迁移一段连续的槽位（`stride`），默认每个线程处理 **16 个槽位**（可通过 `MAX_RESIZERS` 调整）。  
   - 槽位分配通过 `transferIndex` 指针从后向前划分（例如原数组长度 16，线程 A 领取 `[15, 0]`，线程 B 领取 `[31, 16]`）。  

2. **单槽位迁移逻辑**  
   - **单节点情况**：直接复制到新数组对应位置。  
   - **链表情况**：  
     1. 遍历链表，按 `hash & oldCap` 拆分（`0` 留在原位置，`1` 移至 `原位置 + oldCap`）。  
     2. 例如原数组长度 `oldCap=16`，`hash & 16` 为 `0` 时留在槽位 `i`，为 `1` 时移至槽位 `i+16`。  
   - **红黑树情况**：先转为链表再拆分，若拆分后长度 ≥ 6 则重新转为红黑树。  

3. **迁移标记（`ForwardingNode`）**  
   - 槽位迁移完成后，在原数组对应位置放置 `ForwardingNode`（`hash=-1`），用于标识该槽位已迁移至新数组。  


#### 4. 线程协作与退出机制  
1. **线程加入扩容**  
   - 其他线程发现 `sizeCtl < 0` 时，主动调用 `helpTransfer()` 协助扩容：  
     1. 通过 CAS 增加 `sizeCtl` 低 16 位计数（表示新增一个协助线程）。  
     2. 分配未处理的槽位范围，执行迁移。  

2. **线程退出判断**  
   - 线程完成任务后，通过 CAS 减少 `sizeCtl` 计数：  
   ```java
   if (U.compareAndSetInt(this, SIZECTL, sc, sc - 1)) {
       // 若当前线程是最后一个退出的，执行收尾工作
       if ((sc - 2) == (rs << RESIZE_STAMP_SHIFT)) {
           // 触发扩容完成逻辑
       }
   }
   ```  


#### 5. 扩容收尾阶段  
1. **切换 `table` 指针**  
   - 将 `table` 指向新数组，旧数组不再使用（由 GC 回收）。  

2. **更新 `sizeCtl`**  
   - 设置新的扩容阈值为新数组长度的 **3/4**：  
   ```java
   sizeCtl = (newCap << 1) - (newCap >>> 1); // 等价于 newCap * 3/4
   ```  


#### 6. 扩容期间的读写策略  
| 操作类型       | 处理逻辑                                                                 |  
|----------------|--------------------------------------------------------------------------|  
| **读操作**     | - 若槽位存在 `ForwardingNode`，直接访问新数组。<br>- 否则访问原数组。     |  
| **写操作**     | - 若发现正在扩容，先协助迁移部分槽位，再在新数组对应位置执行写入。         |  


#### 7. 核心设计优势  
1. **分段迁移**：将扩容任务拆分为多个小段，多线程并行处理，避免单线程长时间阻塞。  
2. **无锁化设计**：通过 `CAS` 和 `ForwardingNode` 实现读写并发，扩容期间不阻塞正常操作。  
3. **渐进式完成**：部分槽位迁移完成即可提供服务，最终所有线程协作完成整体扩容。  


#### 总结流程图  
```plaintext
触发扩容 → 生成标记 & 创建新数组 → 多线程分段迁移槽位 → 放置ForwardingNode标记 → 线程退出计数 → 最后线程切换table指针 → 扩容完成
```