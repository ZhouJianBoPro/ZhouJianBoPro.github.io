---
layout: post
title: ThreadLocal原理
date: 2022-09-19
tags: [并发编程]
---

#### ThreadLocal特性
ThreadLocal在多线程共享变量时能够避免线程安全问题。它采用的是空间换时间的思想，ThreadLocal提供了线程内存储变量的能力，
即为每个线程提供一份存储空间，线程内存储变量的副本。相比于synchronized，ThreadLocal不会存在锁等待的造成线程阻塞

#### ThreadLocalMap
ThreadLocal中定义了静态内部类ThreadLocalMap，每个线程只持有一个ThreadLocalMap类型的实例threadLocals。
ThreadLocalMap中维护了一个Entry数组，ThreadLocal确定了数组中的一个索引（ThreadLocal hashcode与数组长度取余运算），通过该索引从数组中找到value

- ThreadLocal静态内部类ThreadLocalMap，维护一个Entry数据，用于存储ThreadLocal线程变量
    ```java
    static class ThreadLocalMap {
      
            private static final int INITIAL_CAPACITY = 16;
              
            private Entry[] table;
  
            static class Entry extends WeakReference<ThreadLocal<?>> {
                /** The value associated with this ThreadLocal. */
                Object value;
    
                Entry(ThreadLocal<?> k, Object v) {
                    super(k);
                    value = v;
                }
            }
    }
    ```
- 线程中定义了ThreadLocalMap变量threadLocals
    ```java
    public
    class Thread implements Runnable {
    
        /* ThreadLocal values pertaining to this thread. This map is maintained
         * by the ThreadLocal class. */
        ThreadLocal.ThreadLocalMap threadLocals = null;
    }
    ```
  
#### ThreadLocal#set()
```java
public class ThreadLocal<T> {
    
    public void set(T value) {
        //获取当前线程
        Thread t = Thread.currentThread();
        //获取线程成员变量threadLocals
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

    static class ThreadLocalMap {

        private Entry[] table;

        private void set(ThreadLocal<?> key, Object value) {
    
            Entry[] tab = table;
            int len = tab.length;
            //获取ThreadLocal在ThreadLocalMap数组中的下标
            int i = key.threadLocalHashCode & (len-1);
    
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
    
                if (k == key) {
                    e.value = value;
                    return;
                }
    
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
    }
}
```
int i = key.threadLocalHashCode & (len-1);获取ThreadLocal在ThreadLocalMap数组中的下标，按位与计算逻辑，
相当于取余运算int i = key.threadLocalHashCode % len; 比如threadLocalHashCode = 31， len = 16， 此时i = 15，
011111 & 001111 = 001111 = 15

#### ThreadLocal使用场景示例
基于AbstractRoutingDataSource实现多数据源配置时，对数据源类型进行管理
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

