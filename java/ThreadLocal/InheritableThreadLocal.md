###  InheritableThreadLocal

[TOC]

##### 一、简介

​	在`Thread`中除了有属性`threadLocals`引用`ThreadLocal.ThreadLocalMap`，其实还有一个属性，也就是`inheritableThreadLocals`，`threadLocals`的作用是保存本地线程变量，而`inneritableThreadLocals`的作用是传递当前线程本地变量`InheritableThreadLocal`到子线程的本地变量`InheritableThreadLocal`中。

##### 二、实例

```java
public static void main(String[] args) throws InterruptedException {
  InheritableThreadLocal<String> username = new InheritableThreadLocal<>();
  ThreadLocal<String> password = new ThreadLocal<>();
  username.set("zhangShang");
  password.set("123456789");

  new Thread(new Runnable() {
    @Override
    public void run() {
      System.out.println(username.get());
      System.out.println(password.get());
    }
  }).start();
}
```

输出结果为：

```java
zhangShang
null
```

所以基本上可以得出结论：`InheritableThreadLocal`是具有父子线程传递的，而`ThreadLocal`不具有父子线程传递的功能。

##### 三、原理

###### 1. `InheritableThreadLocal`的实现

`InheritableThreadLocal`继承于`ThreadLocal`，并重写了`ThreadLocal`中的三个方法。

```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    /**
     * 这个接口是ThreadLocal的开放接口，默认实现是抛出UnsupportedOperationException异常。
     * 实现上仅返回入参，调用上是在创建子线程时使用。
     */
    protected T childValue(T parentValue) {
        return parentValue;
    }

    /**
     * 重写getMap,操作InheritableThreadLocal时，将只会影响到线程对象Thread的
     * inheritableThread属性。
     */
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

    /**
     * 与上面的获取方法getMap情况一致，创建时同理。
     */
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

###### 2.线程的创建过程

跟踪`new Thread()`方法。

> 1.进入初始化方法。

```java
public Thread() {
  init(null, null, "Thread-" + nextThreadNum(), 0);
}
```

> 2.调用`init`方法。

```java
// 重载方法
private void init(ThreadGroup g, Runnable target, String name,long stackSize) {
  init(g, target, name, stackSize, null, true);
}

// 最后实际调用方法
private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize, AccessControlContext acc,
                  boolean inheritThreadLocals) {
  if (name == null) {
    throw new NullPointerException("name cannot be null");
  }
  this.name = name;
  // parent线程为创建子线程的当前线程，也就是父线程。
  Thread parent = currentThread();
  
		... 省略一些与本章无关代码
  // inheritThreadLocals=true,默认值是true,且父线程的inheritableThreadLocal对象不为空
  // 创建当前线程的inheritableThreadLocals对象。    
  if (inheritThreadLocals && parent.inheritableThreadLocals != null)
    this.inheritableThreadLocals =
    ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
  /* Stash the specified stack size in case the VM cares */
  this.stackSize = stackSize;

  /* Set thread ID */
  tid = nextThreadID();
}
```

> 3.进入创建方法`createInheritedMap`方法。

```java
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
  // 以父线程的inheritableThreadLocals为实例创建一个ThreadLocalMap对象
  return new ThreadLocalMap(parentMap);
}

/**
 * 以父线程的inheritableThreadLocals为实例创建子线程的inheritableThreadLocals对象
 * 实现上比较简单，将父线程的inheritableThreadLocals循环拷贝给子线程。
 */
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
        // 至于这个地方为什么采用key.childValue(),内层的逻辑也仅仅是返回入参。
        // 网上有些人说是为了减轻代码的阅读难度，笔者觉得有点牵强。感觉是为了在获取
        // 过程中做一些小转换之类的？
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

##### 四、思考

​	上面说父子进程通过`inheritableThreadLocals`属性来传递本地变量，在实际的应用场景中，一般不会出现父进程直接创建子进程的情况，一般都是采用线程池的方式，如果采用线程池那么`inheritableThreadLocal`还会有效吗？读者可以考虑一下，写个`demo`跑一下，看看具体的情况，下一篇文章将进行解答。





