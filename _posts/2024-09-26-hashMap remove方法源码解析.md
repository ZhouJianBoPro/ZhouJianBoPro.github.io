---
layout: post
title: hashMap remove方法解析
date: 2024-09-26
tags: [Map]
---

### remove方法源码
```java
public V remove(Object key) {
        Node<K,V> e;
        // 先计算出key的hash值，然后再移除节点并返回key对应的value
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
}

final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    // 计算hash值对应数组下标并找到对应的元素
    if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        // 数组下标中的元素的key相同，取出该元素
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        // 说明下标元素为红黑树或链表结构
        else if ((e = p.next) != null) {
            // 树结构，从红黑树中取出key对应的元素
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
            // 链表结构，遍历链表，找到key对应的元素
                do {
                    if (e.hash == hash &&
                            ((k = e.key) == key ||
                                    (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        
        // node 为key对应的元素
        if (node != null && (!matchValue || (v = node.value) == value ||
                (value != null && value.equals(v)))) {
            // 当node为树结构时，从红黑树中将该元素移除
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            // 要删除的节点即不是链表结构也不是树结构，下标中只有这一个元素，以该元素的后继元素节点作为下标的值，为null
            else if (node == p)
                tab[index] = node.next;
            // node为链表结构，将要删除元素的前驱元素节点指向要删除元素的后继元素节点
            else
                p.next = node.next;
            ++modCount;
            // hashMap元素数量减一
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

### remove方法执行过程
1. 先计算要删除key的hash值
2. 通过hash值计算出key在数组中的下标，并取出该下标的元素
3. 当key与数组下标元素的key相同时，取出该元素，认为是要删除的元素
4. 当下标元素为红黑树结构时，遍历红黑树并找到key对应的元素
5. 当下标元素为链表结构时，遍历链表并找出key对应的元素
6. 要删除的元素不为null时，当该元素不是红黑树或链表结构，直接将该元素的后继元素节点作为数据下标元素，肯定为null
7. 当要删除的元素为红黑树结构时，从红黑树中移除该元素
8. 当要删除的元素为链表结构时，将该元素的前驱元素节点执行该元素的后继元素节点
9. 在找到要删除元素的情况下，hashMap元素数量减一，并返回删除元素的值