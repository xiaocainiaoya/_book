### ThreadLocal初探

[TOC]

####  一、ThreadLocal的实现原理

​	`Thread`有一个内部变量`ThreadLocal.ThreadLocalMap`,这个类是`ThreadLocal`的静态内部类，它的实现与`HashMap`类似，当线程第一次调用`ThreadLocal`的`get/set`方法时会初始化它。它的键是这个`ThreadLocal`对象本身，值是需要存储的变量。也就是说`ThreadLocal`类型的本地变量是存放在具体的线程空间里。当不断的使用`get`方法获取时，是到线程独有线程空间中获取变量，使得其他线程无法访问到，也就达到了线程安全的目的。在使用完成之后，可以通过`remove`方法，移除不使用的本地变量。

**ThreadLocal和同步机制的比较**

​	如果说同步机制是一种以时间换空间的做法，那么`ThreadLocal`就是一种以空间换时间的做法，在同步机制下，当访问共享变量时，同步机制保证了同一个时刻只能有一个线程访问到，其他线程进入阻塞。`ThreadLocal`下，为每个线程都复制了共享变量的副本，也就不存在共享变量的说法。

#### 二、源码

##### 1.set()

> 通过`ThreadLocal`的set方法调用到`ThreadLocal.ThreadLocalMap`静态内部类的set方法。

```java
public class ThreadLocal<T> {
  public void set(T value) {
          Thread t = Thread.currentThread();
          ThreadLocalMap map = getMap(t);
          if (map != null)
              map.set(this, value);
          else
              createMap(t, value);
  }
  // 若对象为空，则初始化threadLocalMap对象
  void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
  }
  
  static class ThreadLocalMap {
    private void set(ThreadLocal<?> key, Object value) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
						// 先通过hashcode作为下标取数组对应位置的值，若为空，设置值。
      			// 若不为空，往后移动一个位置，如果获取到的长度等于数组长度，从0位置查找。
            for (Entry e = tab[i];e != null;e = tab[i = nextIndex(i, len)]) {
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
     				// 清除Entry对象还在，但是Entry的值为空的位置 && 当前数量是否大于容量
      			// 扩容
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
    }
    // 这边的扩容有两个步骤
    private void rehash() {
      //1. 重新排列table数组里的值，根据hashcode获取下标，若对应下标为空，则移动到该位置
      // 若下标位置不为空，往后移动位置，直到找到空位置。
      expungeStaleEntries();
      // 2.排列的同时如果是空位置，会相应减少size，若排列之后的size仍然大于容量的3/4则扩容
      if (size >= threshold - threshold / 4)
        // 两倍原长度扩容
      resize();
    }
  }
}
```

##### 2.get()

```java
public class ThreadLocal<T> {
  public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
      ThreadLocalMap.Entry e = map.getEntry(this);
      if (e != null) {
        @SuppressWarnings("unchecked")
        T result = (T)e.value;
        return result;
      }
    }
    // 默认值
    return setInitialValue();
  }
  static class ThreadLocalMap {
    private Entry getEntry(ThreadLocal<?> key) {
      int i = key.threadLocalHashCode & (table.length - 1);
      Entry e = table[i];
      if (e != null && e.get() == key)
        return e;
      else
        return getEntryAfterMiss(key, i, e);
    }
    // 获取不到值有两种情况，e=null, e.get() == null,如果e=null直接返回null,
    // 如果e.get()=null，清除这个位置的值。
    private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
      Entry[] tab = table;
      int len = tab.length;

      while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
          return e;
        if (k == null)
          expungeStaleEntry(i);
        else
          i = nextIndex(i, len);
        e = tab[i];
      }
      return null;
    }
  }
}
```

#### 三、继承性

> `ThreadLocal`对父子线程也是一样，同样是不可相互访问。但是，在特殊情况下或许还是存在父子线程需要相互访问`ThreadLocal`中的值的业务需求。可以使用`ThreadLocal`的子类`InheritableThreadLocal`,它可以实现父子进程之间的变量获取。

```java
/**
	InheritableThreadLocal继承ThreadLocal方法，并重写了三个方法，getMap、createMap方法
	是为了让创建map和获取map的时候使用thread中的inheritableThreadLoca变量。而childValue
	是为了在thread父进程调用init创建子进程时，创建子进程的inheritableThreadLocal的时候，逐
	个拷贝父进程的nheritableThreadLocal值。
**/
public class InheritableThreadLocal<T> extends ThreadLocal<T> {

    protected T childValue(T parentValue) {
        return parentValue;
    }

    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}

public class Thread implements Runnable {
      private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
        if (name == null) {
            throw new NullPointerException("name cannot be null");
        }

        this.name = name;

        Thread parent = currentThread();
        SecurityManager security = System.getSecurityManager();
        if (g == null) {
            /* Determine if it's an applet or not */

            /* If there is a security manager, ask the security manager
               what to do. */
            if (security != null) {
                g = security.getThreadGroup();
            }

            /* If the security doesn't have a strong opinion of the matter
               use the parent thread group. */
            if (g == null) {
                g = parent.getThreadGroup();
            }
        }

        /* checkAccess regardless of whether or not threadgroup is
           explicitly passed in. */
        g.checkAccess();

        /*
         * Do we have the required permissions?
         */
        if (security != null) {
            if (isCCLOverridden(getClass())) {
                security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
            }
        }

        g.addUnstarted();

        this.group = g;
        this.daemon = parent.isDaemon();
        this.priority = parent.getPriority();
        if (security == null || isCCLOverridden(parent.getClass()))
            this.contextClassLoader = parent.getContextClassLoader();
        else
            this.contextClassLoader = parent.contextClassLoader;
        this.inheritedAccessControlContext =
                acc != null ? acc : AccessController.getContext();
        this.target = target;
        setPriority(priority);
        // 在这里拷贝
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        this.stackSize = stackSize;
        tid = nextThreadID();
    }
}

// threalLocal中根据threadLocalMap创建threadLocalMap
private ThreadLocalMap(ThreadLocalMap parentMap) {
  Entry[] parentTable = parentMap.table;
  int len = parentTable.length;
  setThreshold(len);
  table = new Entry[len];

  for (int j = 0; j < len; j++) {
    Entry e = parentTable[j];
    if (e != null) {
      @SuppressWarnings("unchecked")
      ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
      if (key != null) {
        // 重写childChild方法
        Object value = key.childValue(e.value);
        Entry c = new Entry(key, value);
        int h = key.threadLocalHashCode & (len - 1);
        while (table[h] != null)
          h = nextIndex(h, len);
        table[h] = c;
        size++;
      }
    }
  }
}
```

#### 四、内存泄漏

![](https://gitee.com/jiangjiamin/image-bed/raw/master/upic/2020-10/xCD8sO.png)

​	`ThreadLcal`的引用关系如上如所示，虚线是使用软引用的地方。如果这个地方使用的是强引用，在业务代码中使用`threadlocalInstance==null`将`ThreadLocalRef`和`ThreadLocal`之间的强引用置空，`value`还是会通过另一条引用链`currentThread->currentThread->map->entry->value`到达，也是不会被GC掉。而若采用软引用，在系统将要发生内存溢出时会回收掉，也就是会断掉`key`与`ThreadLocal`之间的引用，使得`key=null`。

​	在`ThreadLocal`的实现中，为了避免内存泄漏已经做了很多安全性的控制，在`get()`和`set()`方法中都有相应的处理，通过特定的方式对存在`key=null`的脏`Entry`进行`value=null`的处理，使得`value`的引用链不可达。

 **为什么使用弱引用？**

一是尽管使用强引用也会出现内存泄漏，二是在`ThreadLocal`的生命周期中`set、getEntry、remove`里，都针对键为空的脏`Entry`进行处理。但是尽管如此，在编程过程中，形成一种良好的规范，在使用完`ThreadLocal`后都应该手动调用`remove`方法进行清理。