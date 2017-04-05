---
layout: post
title:  "LinkedHashMap详解"
image: ''
description: ''

tags:
- Java
- 源码
- 数据结构
categories:
- Java 基础篇
---
## **1 特点**
- 一个基于LinkedList实现的Map, 能够按照可预测的顺序进行迭代访问。
- 使用一个双端链表来管理所有的元素，这个链表规定了迭代顺序，一般是Key插入的顺序。
- 一个key重复插入时，不会影响它的迭代顺序（保持第一次插入时的顺序？）。
- LinkedHashMap还可以实现按照访问顺序进行迭代访问，类似LRU缓存。调用put,get,getOrDefault,compute, putIfAbsent,computeIfPresent或者merge等方法都会对某个entry进行访问。replace()方法只会在entry的value真正被替换的时候才算进行了访问。putAll()对一个特定map的所有映射都进行了一次entry access，access顺序和特定map迭代器提供的访问属性一致。
- 可以重新removeEldestEntry()来实现添加一个新映射到Map时，就删除最老的entry的功能。
- 当不发生hash冲突时，提供O(1)的add,contains和remove。
- 由于要维护链表，性能可能略低于HashMap，除了遍历时，LinkedHashMap时间只跟size有关，而HashMap跟capactiy也有关。
- 由两个参数影响HashMap的性能：
    - initial capacity：capacity是HashMap桶的数量，不要求是2的倍数，默认为11
    - load factor：规定在HashMap达到多少使用率时进行自动扩容成原来桶的2倍。
- 结构的修改包括：任何删除或添加映射的方法，在access-ordered LinkedHashMap中影响了迭代顺序。注意，在Insertion-ordered中修改某个映射的value不算结构的修改。
- fail-fast。

## **2 实现**
 
![image](/images/lhm.png)

LinkedHashMap继承关系如上图所示，它继承了HashMap，存储的每个entry都比HashMap.Node多了两个指向指针来维护一个双向链表。
```java
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```
调用LinkedHasmap实例的put()方法会直接调用HashMap的put()方法，不过LinkedHashMap覆盖了newNode()来使Map中的所有entry不但存储在一个桶数组中，还构成了一个双向链表。
```java
// HashMap的putVal()方法，调用LinkedHashMap实例的put方法最终也会调用这个方法
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
        //************* LinkedHashMap覆盖了newNode方法，会返回一个LinkedHashMap.Entry***********//
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
                // 在访问某个entry后调用，用来在access-order模式下更新链表中的顺序。
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        // 在LinkedHashMap插入entry之后调用，主要用来删除最老（插入顺序或访问顺序）的entry.
        afterNodeInsertion(evict);
        return null;
    }
```
LinkedHashMap的newNode方法实现如下，可以看到该方法首先创建了一个LinkedHashMap.Entry实例，然后将这个entry实例加入到链表末尾。
```java
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }
    private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
        LinkedHashMap.Entry<K,V> last = tail;
        tail = p;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
    }
```
在HashMap的get方法中，还留下了2个钩子能够让LinkedHashMap在插入或者访问entry时做一些后置的处理，就是afterNodeInsertion()和afterNodeAccess()方法：
- 在afterNodeInsertion()方法中，调用removeEldestEntry()方法来实现额外的逻辑（比如删除最旧的元素等）。
- 这afterNodeAccess()方法中，如果LinkedHashMap元素排序策略是按照访问顺序，就需要在每次元素被访问之后，调整顺序。

```lava
 void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        // 如果removeEldestEntry判断最老的entry需要删除，那么就删除这个entry
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }

    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        // 如果是access-order模式其e不是链表尾节点（尾节点是最新的节点，迭代时最先被访问）就将e移动到链表最后面。
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
```
get()方法也是直接使用HashMap的实现
```java
    public V get(Object key) {
        Node<K,V> e;
        // 调用HashMap.getNode，从桶数组中找到对应key的value
        if ((e = getNode(hash(key), key)) == null)
            return null;
        // 如果是access-order，还要进行链表顺序的更新
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }
```

可以看到，LinkedHashMap不仅通过桶数组来存储所有entry，从而保证o(1)的查找时间，也通过维护一个所有entry的双端链表来保证按插入顺序或访问顺序来遍历集合。

和HashMap不同LinkedHashMap的Iterator都是对双向链表的迭代，迭代器实现如下。
```java
abstract class LinkedHashIterator {
    LinkedHashMap.Entry<K,V> next; // 下一个元素
    LinkedHashMap.Entry<K,V> current; // 当前元素
    int expectedModCount;

    LinkedHashIterator() {
        next = head;
        expectedModCount = modCount;
        current = null;
    }

    public final boolean hasNext() {
        return next != null;
    }
    
    final LinkedHashMap.Entry<K,V> nextNode() { // 返回链表下一个元素
        LinkedHashMap.Entry<K,V> e = next;
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (e == null)
            throw new NoSuchElementException();
        current = e;
        next = e.after;
        return e;
    }

    public final void remove() {
        Node<K,V> p = current;
        if (p == null)
            throw new IllegalStateException();
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        current = null;
        K key = p.key;
        removeNode(hash(key), key, null, false, false);
        expectedModCount = modCount;
    }
}

final class LinkedKeyIterator extends LinkedHashIterator
    implements Iterator<K> {
    public final K next() { return nextNode().getKey(); }
}

final class LinkedValueIterator extends LinkedHashIterator
    implements Iterator<V> {
    public final V next() { return nextNode().value; }
}

final class LinkedEntryIterator extends LinkedHashIterator
    implements Iterator<Map.Entry<K,V>> {
    public final Map.Entry<K,V> next() { return nextNode(); }
}

```