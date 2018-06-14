---
layout:     post
title:      ThreadLocal 源码
subtitle:   ""
date:       2018-06-14
author:     Logic
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - ThreadLocal
    - 多线程
    - 源码
---
ThreadLocal类能使线程中的某个值与保存值的对象关联起来，每个使用该变量的线程都存有一份独立的副本。

### 成员变量

- `private final int threadLocalHashCode` 
- `private static AtomicInteger nextHashCode =    new AtomicInteger();` 
- `private static final int HASH_INCREMENT = 0x61c88647` 每次hash值的增量

### 方法

#### `initialValue()` 方法

```java
protected T initialValue() {
    return null;
}
```

该方法在每个可以为`ThreadLocal`变量赋初始值，一般在创建`ThreadLocal`对象的时候重写该方法。默认情况下返回为`null`。

#### `get()` 方法

```java
public T get() {
    //获取当前运行的线程
    Thread t = Thread.currentThread();
    //从当前的Thread中获取ThreadLocalMap对象，每个Thread都包含一个ThreadLocal对象
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        //从ThreadLocalMap中获得当前ThreadLocal中的值
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    //如果前面条件都不满足，返回初始值
    return setInitialValue();
}
```

该方法首先获取当前运行的线程，然后从当前的`Thread`中获得`ThreadLocalMap`对象，然后从`ThreadLocalMap`中获得当前`ThreadLocal`变量所对应的值。

#### `ThreadLocalMap getMap(Thread t)` 方法

```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

该方法直接从`Thread`中获得由`Thread`维护的一个`ThreadLocalMap`对象。由于`Thread`类与`ThreadLocal`类位于同一个包中，并且在`Thread`类中`ThreadLocal.ThreadLocalMap threadLocals` 的作用域为包作用域，所以在`getMap()` 方法中直接利用`t.threadLocals` 获得`Thread t` 中的`ThreadLocalMap`对象。

#### `set(T value)` 方法

```java
public void set(T value) {
    //获得当前线程
    Thread t = Thread.currentThread();
    //获得当前线程的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    //判断当前线程的ThreadLocalMap是否已经被初始化
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

`set`方法是为当前线程的`ThreadLocal`变量赋值。首先获得当前线程，然后获得当前线程的`ThreadLocalMap`对象，判断当前线程的`ThreadLocalMap`对象是否已经被初始化，如果已经被初始化，则赋值。如果未被初始化，则将当前线程的`ThreadLocal.ThreadLocalMap threadLocals` 变量进行初始化，然后再赋值。

#### `createMap(Thread t, T firstValue)` 方法

```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

在创建创建或者启动一个线程的时候，不会对`Thread` 类中的`threadLocals` 变量进行初始化，只有用到`ThreadLocal` 变量，以及第一次进行赋值的时候，才会将`Thread` 类中的`threadLocals` 变量进行初始化。

### ThreadLocalMap类 

#### 成员变量

- `private static final int INITIAL_CAPACITY=16` `ThreadLocalMap`的起始大小，必须是2的整数倍
- `private Entry[] table` 用来存放`ThreadLocal`变量相对应的值
- `private int size` `ThreadLocalMap`的大小
- `private int threshold` 扩容时的大小

#### 内部类

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

针对`ThreadLocalMap`，期包含一个`Entry`类型的数组`table`,` Entry`类继承`WeakReference`，便于垃圾回收的时候进行处理。

#### 方法

##### 构造方法

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    //对table数组进行初始化
    table = new Entry[INITIAL_CAPACITY];
    //根据firstKey的threadLocalHashCode找到应该存放firstValue的位置
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    //创建Entry对象，并赋值给table[i]
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

首先对`table` 数组进行初始化，然后根据`firstKey`的`threadLocalHashCode`找到应该存放`firstValue`的位置，最后创建`Entry`对象，并赋值给`table[i]`。

##### `getEntry(ThreadLocal<?> key)` 方法

```java
private Entry getEntry(ThreadLocal<?> key) {
    //计算出hash位置
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    //判断当前位置为空或者当前位置值为空
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```

首先根据`ThreadLocal`的`threadLocalHashCode`计算出在`table`数组中的位置，然后获取该位置上的数据，如果该位置上的数据不为空，则返回当前位置数据，如果为空，调用`getEntryAfterMiss()` 方法。

##### `getEntryAfterMiss` 方法

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;
    //如果e==null的话，说明没有该值，直接返回null,
    //如果e!=null,说明找对了，或者值已经过期，或者冲突
    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            //如果Entry对象存在，而获得不了值，则说明值已经过期，则调用expungeStaleEntry()方法首先进行rehash,等rehash结束之后，再反过来继续查找
            expungeStaleEntry(i);
        else
            //如果Entry对象存在，并且当前的key不等于要查找的key，说明发生冲突，则需要向下加1查找
            i = nextIndex(i, len);
        e = tab[i];
    }
    //如果都查找不到，则返回null
    return null;
}
```

##### `expungeStaleEntry(int staleSlot)` 方法

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    //删除staleSlot位置上的元素
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    //对table数组进行rehash，直到遇到null为止
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        //如果在第i个位置上，e!=null，而k==null，说明该位置元素已过期
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            //如果h!=i说明不是直接hash过来的值，而是利用线程冲突解决方法解决的
            if (h != i) {
                tab[i] = null;

                // 给元素e重新hash一个位置
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```



##### `set(ThreadLocal<?> key, Object value)` 方法

```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    //通过hashCode定位在table中的位置
    int i = key.threadLocalHashCode & (len-1);
    //从第i个位置向后遍历，找到当前key所对应value的位置
    //如果第i个位置为null的话，就直接省略循环
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }
        //如果当前位置的值已经被删除，则调用replaceStaleEntry方法
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

在设置值的时候，首先计算出当前`key`在`table`数组中的位置，如果当前位置为`null`的话，则直接给`table[i]`赋值，如果不为空，则说明`key`已经存在，或者发生冲突，利用`for`循环查找设置值的位置。如果找到对应的`key`，则直接设置值。如果对应的`e.get()==null`，说明其值被删除，则调用`replaceStaleEntry`方法设置当前位置的值。

##### `replaceStaleEntry()` 方法

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                               int staleSlot) {
  Entry[] tab = table;
  int len = tab.length;
  Entry e;

  //从staleSlot位置向前查找第一个非失效数据的位置
  int slotToExpunge = staleSlot;
  for (int i = prevIndex(staleSlot, len);
       (e = tab[i]) != null;
       i = prevIndex(i, len))
    if (e.get() == null)
      slotToExpunge = i;

  // 从staleSlot位置向后查找key所对应元素的位置
  for (int i = nextIndex(staleSlot, len);
       (e = tab[i]) != null;
       i = nextIndex(i, len)) {
    ThreadLocal<?> k = e.get();

    //如果找到了对应的key，则替换当前key所对应的value值
    if (k == key) {
      e.value = value;

      tab[i] = tab[staleSlot];
      tab[staleSlot] = e;

      // Start expunge at preceding stale entry if it exists
      if (slotToExpunge == staleSlot)
        slotToExpunge = i;
      cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
      return;
    }

    // If we didn't find stale entry on backward scan, the
    // first stale entry seen while scanning for key is the
    // first still present in the run.
    if (k == null && slotToExpunge == staleSlot)
      slotToExpunge = i;
  }

  // If key not found, put new entry in stale slot
  tab[staleSlot].value = null;
  tab[staleSlot] = new Entry(key, value);

  // If there are any other stale entries in run, expunge them
  if (slotToExpunge != staleSlot)
    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

##### `ThreadLocalMap`里`Entry`为何声明为`WeakReference`类型？

WeakReference

> WeakReference是Java语言规范中为了区别直接的对象引用（程序中通过构造函数声明出来的对象引用）而定义的另外一种引用关系。WeakReference标志性的特点是：reference实例不会影响到被应用对象的GC回收行为（即只要对象被除WeakReference对象之外所有的对象解除引用后，该对象便可以被GC回收），只不过在被对象回收之后，reference实例想获得被应用的对象时程序会返回null。

#### 使用T`hreadLocal`的建议

1. `ThreadLocal`类变量因为本身定位为要被多个线程来访问，它通常被定义为`static`变量。
2. 能通过值传递的参数，不用通过`ThreadLocal`存储，以免造成`ThreadLocal`滥用。
3. 在线程池情况下，在`ThreadLocal`业务周期处理完成时，最好显示的调用`remove()`方法，清空县城关局部变量中的值。
4. 在正常情况下使用`ThreadLocal`不会造成`OOM`, 弱引用的知识`ThreadLocal`,保存值依然是强引用，如果`ThreadLocal`依然被其他对象应用，线程局部变量将无法回收。