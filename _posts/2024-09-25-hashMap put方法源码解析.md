---
layout: post
title: hashMap put方法源码解析
date: 2024-09-25
tags: [hash]
---

### put方法源码
```java
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
}

// 计算键值的hashCode，用于确定元素在数组中的下标
static final int hash(Object key) {
    int h;
    // 1. 当key为null时，hash值为0，即元素在数组中的下标为0
    // 2. 异或运算右移16位可以提高hash值的均匀性，减少哈希碰撞的概率
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// onlyIfAbsent 在put中为false， 在putIfAbsent中为true
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    
    // 数组 & 前驱元素
    Node<K,V>[] tab; Node<K,V> p;
    
    // 数组长度 & 元素在数组中的下标
    int n, i;
    
    // 数组为空时调用扩容方法resize初始化数组
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    // i = (n - 1) & hash 通过位与运算计算出元素对应hash值在数组中的下标，等同于hash % n，位与运算效率更高
    // 如果直接用n进行位与运算，假设初始化数组长度为16，二进制码为0001 0000（省略前面24位），会导致与其他任意hash值位与运算后得出的下标要么为0要么为16
    // p为该数组下标已存在的元素，可以称为前驱元素
    if ((p = tab[i = (n - 1) & hash]) == null)
        // 该数组下标没有元素，创建一个新元素放入该下标中
        tab[i] = newNode(hash, key, value, null);
    else {
        // 该数组下标已经存在元素（发生哈希碰撞）
        Node<K,V> e; K k;
        
        // 插入元素键值 = 前驱元素键值时（条件是hash值与键值必须都相等）， 现有元素 = 前驱元素
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            // 前驱元素为红黑树结构
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 前驱元素为链表结构时，遍历链表，binCount为统计链表的长度
            for (int binCount = 0; ; ++binCount) {
                // 当前驱元素指向的下一个元素不存在时，创建一个新元素放入该链表末尾
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 当链表长度达到8时，转换为红黑树（同时要满足数组长度>64，否则再次进行扩容将长链表改为两个短链表存储在新数组中）。而且不包含本次插入的元素
                    if (binCount >= TREEIFY_THRESHOLD - 1) 
                        // 调用treeifyBin方法将链表转换为红黑树结构
                        treeifyBin(tab, hash);
                    break;
                }
                // 遍历链表时，发现要新增的键值已经存在了，则退出循环
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                
                p = e;
            }
        }
        
        // 现有元素存在时，表示键值已存在，则返回旧值
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // 是否覆盖旧值逻辑：putIfAbsent方法onlyIfAbsent为true，不覆盖旧值 （但是旧值为null时也会覆盖）
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    
    // 当元素新增时（key不存在），对hashMap中的size进行累加并再次检查扩容，n = 2 * n， 这样扩容后得到的（n - 1）转换为2进制低位都为1
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}

final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    // 
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        // 扩容
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```

### put方法执行过程
1. 先计算元素键值的hash值，(h = key.hashCode()) ^ (h >>> 16) 
2. 判断数组是否需要进行初始化，如果数组为空时，需要调用resize方法进行初始化
3. 通过hash值计算出元素在数组中的下标，（n - 1）& hash = hash % n
4. 检查数组在该下标中是否存在元素，不存在时创建一个新元素放入到该下标中
5. 当该下标存在元素时（哈希冲突），首先判断已存在的元素键值是否与当前键值相等（hash值与键值都相等）。相等时直接修改值，并且返回返回旧值，程序结束
6. 反之再判断下标已存在的元素是否为红黑树结构，如果是的话将元素插入到红黑树中
7. 以上情况都不满足，说明该元素应该是链表结构，遍历链表。当链表中存在该键值时，只需修改值即可，并且返回旧值，程序结束
8. 当遍历完链表发现键值不存在时，创建一个新的元素放入到链表的尾部。当链表长度大于8（树化阈值）时，同时数组长度大于64（最小树化阈值）时，会将该下标的链表转换为红黑树结构。（数组长度不大于64时），数据会再次进行扩容，将长链表转换为两条短链表分别存放在新数组的低位下标和高位下标
9. put过程中hashMap中没有重复的键值，会累计数组长度，并检查是否需要扩容操作

### 链表头插法与尾插法区别
1. 头插法：在链表头部插入元素，效率高，无需遍历整个链表，时间复杂度为O(1)
2. 尾插法：在链表尾部插入元素，需要遍历整个链表，时间复杂度为O(n), 其中n为链表长度
3. jdk1.7 hashMap链表采用了头插法，而1.8采用的是尾插法
4. 在插入效率上头插法更快，因为无需遍历链表；但是在多线程中，使用头插法可能导致链表闭环；尾插法生成的链表顺序与插入顺序一致，而头插法相反

#### jdk1.7和1.8之间的区别
1. 底层结构：1.7（数组 + 链表），1.8（数组 + 链表 + 红黑树）
2. 元素插入链表：1.7（头插法），1.8（尾插法）

#### 扩容机制
> 扩容触发条件：1. 数组元素达到扩容阈值（0.75 * cap）；2. 链表长度达到树化阈值（8）且数组长度小于最小树化阈值（64）。
> 在拆分红黑树/单向链表为两个链表时，通过（e.hash & oldCap == 0）来区分 低位 还是 高位，注意oldCap不用减1
1. 创建一个新数组，容量为原数组两倍，并遍历旧数组元素
2. 当遍历到的元素没有下一个结点时（e.next == null），将该元素放入新数组中，重新计算下标hash & (newCap - 1))
3. 当遍历到的元素为红黑树时，先将红黑树拆分为两个双向链表（低位 + 高位），低位链表放入新数组中位置（Index），高位链表在新数组中位置（oldIndex + oldCap），再分别将双向链表转换为单向链表或红黑树（通过去树化阈值判断，默认为6）
4. 当数组下标为链表时，将链表分为两个单向链表（低位 + 高位），低位链表放入新数组中位置（Index），高位链表在新数组中位置（oldIndex + oldCap）

#### 并发访问下引发的问题
1. 写入时发生hash冲突时可能会导致值被覆盖
2. jdk1.7及以下使用链表头插法，可能会导致链表形成闭环，在访问时会导致死循环
