---
layout: post
title:  "JAVA并发-ThreadLocal源码解析"
image: ''
description: ''

tags:
- Java
- 多线程
- 源码
categories:
- Java 基础篇
- 学习记录
---
## **1 使用方法**
ThreadLocal用来实现变量的线程封闭，使用ThreadLocal定义的变量在每个线程中都有个独立的副本，ThreadLocal.get()会得到当前线程对应的副本，ThreadLocal.set()会更新当前线程对应的副本。

在创建ThreadLocal时，可以通过覆盖initialValue方法来自定义ThreadLocal变量的初始值。

## **2 实现方法**

### **2.1 ThreadLocal存储方式**

**在Thread类中包含一个ThreadLocal.ThreadLocalMap的实例域threadLocals，这个实例域就类似于一个HashMap，存储了当前线程对应的ThreadLocal变量。**

```java
//Thread

 /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
```
### **2.2 ThreadLocalMap与ThreadLocal的延迟初始化**

Thread实例中存放ThreadLocal的Map和每个ThreadLocal实例都是延迟初始化的，如下代码所示：
```java
    /**
     * Creates a thread local variable.
     * @see #withInitial(java.util.function.Supplier)
     */
    public ThreadLocal() {
    }
    
    public T get() {
        // 得到当前线程
        Thread t = Thread.currentThread();
        // 得到当前线程对应的ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        if (map != null) { // 如果当前线程对应的ThreadLocalMap已经存在
            
            ThreadLocalMap.Entry e = map.getEntry(this); // 得到ThreadLocal变量对应的Entry
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result; // 返回ThreadLocal变量的值
            }
        }
        return setInitialValue();
    }
    
    
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```
从上面可以看到，ThreadLocal.get方法会先得到当前线程所对应的Thread对象，然后从Thread对象的ThreadLocalMap中查找对应的ThreadLocal对象。如果ThreadLocalMap不存在，或者Map中不包含对应的ThreadLocal对象，就执行setInitialValue方法来返回进行初始化。

```java
private T setInitialValue() {
    // 得到ThreadLocal变量的初始值
    T value = initialValue();
    // 得到当前Thread的ThreadLocalMap
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    
    if (map != null)
        map.set(this, value); // Map存在，更新当前ThreadLocal变量对应的值
    else
        createMap(t, value); // 创建Map并插入
    return value;
}

protected T initialValue() {
    return null;
}
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
我们再来看看ThreadLocal.set方法，与get类似，如果ThreadLocalMap不存在就创建Map再插入值，否则直接更新值
```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```
### **2.3 ThreadLocalMap**

现在，我们看到，**ThreadLocal变量其实是存储在每个Thread对象的ThreadLocalMap中的**，调用ThreadLocal的get/set方法，其实最终都是对ThreadLocalMap进行操作。

ThreadLocalMap是ThreadLocal中的一个静态内部类，它其实是一个Hash表的实现，有以下一些特点：
1. **使用了开放寻址法来解决Hash冲突**。
2. **每个Entry的key是WeakReference，而value不是。因此需要额外的去清理因为GC导致key为null的Entry**。

#### **2.3.1 域**

ThreadLocalMap包含的域有以下一些，Entry类表示一个ThreadLocal变量，它继承了WeakReference，value就是ThreadLocal变量的值。
```java
static class ThreadLocalMap {

    /**
     * The entries in this hash map extend WeakReference, using
     * its main ref field as the key (which is always a
     * ThreadLocal object).  Note that null keys (i.e. entry.get()
     * == null) mean that the key is no longer referenced, so the
     * entry can be expunged from table.  Such entries are referred to
     * as "stale entries" in the code that follows.
     */
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }

    /**
     * 初始容量 -- MUST be a power of two.
     */
    private static final int INITIAL_CAPACITY = 16;

    /**
     * The table, resized as necessary.
     * table.length MUST always be a power of two.
     * 存储ThreadLocal的数组
     */
    private Entry[] table;

    /**
     * The number of entries in the table.
     * 数组中包含entries的个数
     */
    private int size = 0;

    /**
     * The next size value at which to resize.
     * 下一次需要进行resize的阈值
     */
    private int threshold; // Default to 0
    
    //其他代码
}
```
### **2.3.2 对无效Entry的清理**

因为ThreadLocalMap.Entry使用ThreadLocal的弱引用作为key，因此可能出现key为null的无效Entry。需要回收这些key为null的entry来防止内存泄漏。回收在expungeStaleEntry()方法中实现。

由于使用了开放寻址法，在回收无效Entry之后，需要对它影响的所有Entry（和被回收的key冲突）进行Rehash。
```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // 回收当前entry
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash接下来的entry，直到遇到null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        // 当前entry的key为null，需要回收
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else { // rehash entry
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```
### **2.3.3 getEntry**

ThreadLocal.get其实就是根据ThreadLocal对象的hashcode从ThreadLocalMap中查找元素，ThreadLocal#get()会调用ThreadLocalMap#getEntry()来得到当前线程对应的ThreadLocal对象。

```java
private Entry getEntry(ThreadLocal<?> key) {
    // 根据ThreadLocal的HashCode来计算它在table数组中的下标
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    // 找到对应的key对应的Entry
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```

ThreadLocalMap使用开放寻址法来解决Hash冲突，具体实现就在getEntryAfterMiss()中。

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;
    
    // 开放寻址
    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null) // 如果table[i]的ThreadLocal为空，需要回收这个失效的Entry，并重新rehash
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len); // 线性探查下一个位置
        e = tab[i];
    }
    return null;
}

private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}
```
### **2.3.4 set**

ThreadLoca.set最终会调用ThreadLocalMap.set， set会使用开放寻址法来找到对应key的entry。

在查找的过程中，可能会碰到无效的Entry（key已经被回收）。这时候需要替换无效的entry。可能key对应的entry在table的后面，这时就需要交换无效entry和key对应entry的位置，并rehash来保证table中的顺序不被打乱，这些都是在replaceStaleEntry方法中实现。
```java
private void set(ThreadLocal<?> key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    // 查找键key的entry
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        // 找到就更新value
        if (k == key) {
            e.value = value;
            return;
        }
        // 如果找到一个无效的entry，替换无效的entry（可能对应key的entry在table后面能找到，就需要进行交换）
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```