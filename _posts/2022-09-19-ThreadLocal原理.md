---
layout: post
title: ThreadLocal原理
date: 2022-09-19
tags: [concurrency]
---

#### ThreadLocal特性
> ThreadLocal是 Java 中用于实现线程本地变量的类， 它的核心作用是 让每个线程拥有独立的ThreadLocal变量，不同线程之间互不干扰。
> ThreadLocal适用于在多线程环境下保存线程上下文信息（如用户信息、事务对象）

#### ThreadLocal原理
> Thread线程中维护了一个ThreadLocalMap变量（threadLocals），而ThreadLocalMap是ThreadLocal中的内部类，
> 它维护了一个Entry数组，每个Entry数组中保存一个ThreadLocal变量和变量对应的值。在ThreadLocal#set过程中，
> 实际上是把变量设置到Thread线程中的ThreadLocalMap中，以ThreadLocal作为键。同时ThreadLocal#get则是从当前线程的ThreadLocalMap中取出变量值

#### ThreadLocalMap
> ThreadLocal中定义了静态内部类ThreadLocalMap，每个线程只持有一个ThreadLocalMap类型的实例threadLocals。 
> ThreadLocalMap中维护了一个Entry数组（初始大小16），ThreadLocal确定了数组中的一个索引（ThreadLocal hashcode与数组长度取余运算），通过该索引从数组中找到value

```java
// Thread线程中维护了一个ThreadLocalMap变量（threadLocals）
public class Thread implements Runnable { 
    ThreadLocal.ThreadLocalMap threadLocals = null;
}

public class ThreadLocal<T> {

    // ThreadLocalMap是ThreadLocal中的内部类
    static class ThreadLocalMap {
    private static final int INITIAL_CAPACITY = 16;
    private Entry[] table;

    // Entry的key是弱引用（自动被GC回收），value是强引用（手动remove回收）
    static class Entry extends WeakReference<ThreadLocal<?>> {
      /** The value associated with this ThreadLocal. */
      Object value;

      Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
      }
    }
  }
}
```
  
#### ThreadLocal#set
1. 获取当前线程中的ThreadLocalMap变量（threadLocals）
2. 计算ThreadLocal作为key在ThreadLocalMap中Entry数组的下标（threadLocalHashCode）
3. 当发生hash冲突时，使用线性探测法（遍历Entry数组下一个位置，直到找到空位）解决hash冲突
4. 未发生hash冲突时，进行添加操作
```java
public class ThreadLocal<T> {
    
    public void set(T value) {
        //获取当前线程
        Thread t = Thread.currentThread();
        //获取线程成员变量threadLocals
        ThreadLocalMap map = t.threadLocals;
        map.set(this, value);
    }

    static class ThreadLocalMap {

        private Entry[] table;

        private void set(ThreadLocal<?> key, Object value) {
    
            Entry[] tab = table;
            int len = tab.length;
            // 1. 获取ThreadLocal在ThreadLocalMap数组中的下标，位与运算（key.threadLocalHashCode % len）
            int i = key.threadLocalHashCode & (len-1);
    
            // 2. 使用线性探测法解决hash冲入：发生hash冲突时，会不断遍历Entry数组下一个位置，直到找到空位
            for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
                // 找到相同key（非hash冲突），更新value。说明该线程已经设置过这个ThreadLocal的值
                if (k == key) {
                    e.value = value;
                    return;
                }
                
                // 弱引用key已被GC回收，对value进行清理和替换操作
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
    
            // 3. 未发生hash冲突，进行添加操作
            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                // 当Entry数组内的元素个数达到阈值（扩容因子 = 2/3），即threshold = tab.length * 2/3
                rehash();
        }
    }
}
```

#### ThreadLocal如何解决hash冲突
> ThreadLocalMap底层是通过Entry数组进行存储的，数组下标为threadLocalHashCode % tab.length。
> 当set/get过程中发生hash冲突时（ThreadLocal不相同但是hashcode相同），采用线性探测法解决hash冲突，
> 即从当前位置，持续遍历下一个位置，直到找到空位

#### ThreadLocal如何扩容
> ThreadLocal扩容因子默认2/3，当Entry数组中的元素个数达到阈值时（tab.length * 2/3），会进行容量翻倍扩容

#### ThreadLocal使用场景示例
1. 基于AbstractRoutingDataSource实现多数据源配置时，对数据源类型进行管理
    ```java
    public class DataSourceTypeManager {
    
        private static final ThreadLocal<DataSourceEnum> dataSourceTypes = new ThreadLocal<DataSourceEnum>() {
            @Override
            protected DataSourceEnum initialValue(){
                return DataSourceEnum.DATASOURCE1;
            }
        };
    
    
        public static DataSourceEnum get(){
            return dataSourceTypes.get();
        }
    
        public static void set(DataSourceEnum dataSourceType){
            dataSourceTypes.set(dataSourceType);
        }
    }
    ```
2. 日期工具类封装（SimpleDateFormat非线程安全）
    ```java
    public class DateUtils {
        
        private static final ThreadLocal<SimpleDateFormat> YYYY_MM_DD = ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
        
        public static String format(Date date) {
            return YYYY_MM_DD.get().format(date);
        }
    }
    ```

