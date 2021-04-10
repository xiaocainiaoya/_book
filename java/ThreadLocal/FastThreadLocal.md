### FastThreadLocal

[TOC]

#### 一、背景

​	因为需要，研究了可以通过`InheritableThreadLocal`进行父子线程中如何传递本地线程变量，通过阿里开源项目`TransmitableThreadLocal`进行进行线程池传递本地线程变量(详解可查看以往博客)。在查找资料的过程中无意发现了`Dobbo`的`InternalThreadLocal`，其实`Dobbo`的`InternalThreadLocal`和`netty`的`FastThreadLocal`有异曲同工之妙。之前学`netty`的时候有了解一点，为了加深一下`ThreadLocal`种群的了解，使用本博客记录一下。

#### 二、实例

```java
public static void main(String[] args) {
	
  /**
   * 原生线程调用方式
   */
  new Thread(new Runnable() {
    @Override
    public void run() {
      username.set("zhangSang");
      password.set("123456");
      password.remove();

      System.out.println(username.get());
      System.out.println(password.get());
    }
  }).start();
	
	/**
   * FastThreadLocalThread线程调用方式
   */
  new FastThreadLocalThread(new Runnable() {
    @Override
    public void run() {
      username.set("zhangSang");
      password.set("123456");

      System.out.println(username.get());
      System.out.println(password.get());
    }
  }).start();
}
```

单从运行结果看，都能获取到对应设置的值，二者没有任何输出区别，但是跟踪一下，可以看到调用的逻辑是有所区别的。

#### 三、原理

​	熟悉`ThreadLocal`的应该知道，`Threadlocal`其实内部逻辑是一个以`ThreadLocal`对象为`key`，需要存储的值为`value`的`map`结构。而`FastThreadLocal`的内部实现是一个数组，实现上是直接通过下标定位元素，所以单纯从取、存的角度看，`FastThreadLocal`是比`ThreadLocal`高效。

​	对于使用原生线程`Thread`来说，其实最后是将数据存`Thread.threadLocals(ThreadLocal.ThreadLocalMap)`中,也就是说在这个线程所使用的所有`FastThreadLocal`最后都以 `key=ThreadLocal<InternalThreadLocalMap>`
对象,`value=InternalThreadLocalMap`的方式存在线程`ThreadLocal`的一个节点中。而若使用的是`netty`封装的`FastThreadLocalThread`,则`FastThreadLocalThread`对象的属性`threadLocalMap`中。

​	`FastThreadLocalThread`是直接继承于线程类`Thread`，并且内部维护一个`InternalThreadLocalMap`，用于存储变量。虽然这个类命名为`Map`，结构上其实是一个数组。并且下标为0的元素是一个`Set<FastThreadLocal<?>>`的结构，存储着当前有效的`FastThreadLocal`。

```java
public class FastThreadLocalThread extends Thread{
  private InternalThreadLocalMap threadLocalMap;
}
```

在`InternalThreadLocalMap`中提供了一些静态方法，通过当前线程的不同类型，以不同的方式获取对应所需要的`InternalThreadlocalMap`。

```java
/**
 * 获取方法
 */
public static InternalThreadLocalMap get() {
  Thread thread = Thread.currentThread();
  // 根据不同的线程类型，以不同的方式获取对应的InternalThreadLocalMap
  if (thread instanceof FastThreadLocalThread) {
    return fastGet((FastThreadLocalThread) thread);
  } else {
    return slowGet();
  }
}

/**
 * FastThreadLocalThred获取方法
 */
private static InternalThreadLocalMap fastGet(FastThreadLocalThread thread) {
  // 直接从FastThreadLocalThread对象中获取InternalThreadLocalMap
  InternalThreadLocalMap threadLocalMap = thread.threadLocalMap();
  // 若为空,初始化一个InternalThreadLocalMap
  if (threadLocalMap == null) {
    thread.setThreadLocalMap(threadLocalMap = new InternalThreadLocalMap());
  }
  return threadLocalMap;
}

/**
 * 原生线程的获取方法
 */
private static InternalThreadLocalMap slowGet() {
 /**
  * 从UnpaddedInternalThreadLocalMap中获取到ThreadLocal<InternalThreadLocalMap>
  * 在从ThreadLocal中获取InternalThreadLocalMap,若为空初始化一个。所以由此可知，原生线程
  * 的FasstThreadLocal具体值，是以InternalThreadLocalMap为值，存储在ThreadLocal中。
  */
  ThreadLocal<InternalThreadLocalMap> slowThreadLocalMap = 
    UnpaddedInternalThreadLocalMap.slowThreadLocalMap;
  InternalThreadLocalMap ret = slowThreadLocalMap.get();
  if (ret == null) {
    ret = new InternalThreadLocalMap();
    slowThreadLocalMap.set(ret);
  }
  return ret;
}
```

##### 1.set值

```java
public final void set(V value) {
  if (value != InternalThreadLocalMap.UNSET) {
    // 根据不同的线程类型，获取到的InternalThreadLocalMap进行设置值。
    set(InternalThreadLocalMap.get(), value);
  } else {
    remove();
  }
}

public final void set(InternalThreadLocalMap threadLocalMap, V value) {
  if (value != InternalThreadLocalMap.UNSET) {
    // 将需要存储的值添加到InternalThreadLocalMap对应的下标位置。
    if (threadLocalMap.setIndexedVariable(index, value)) {
      addToVariablesToRemove(threadLocalMap, this);
    }
  } else {
    remove(threadLocalMap);
  }
}

private static void addToVariablesToRemove(InternalThreadLocalMap threadLocalMap,
                                           FastThreadLocal<?> variable) {
  // 根据下标获取对应的元素
  Object v = threadLocalMap.indexedVariable(variablesToRemoveIndex);
  Set<FastThreadLocal<?>> variablesToRemove;
  if (v == InternalThreadLocalMap.UNSET || v == null) {
    // 若为空，创建set结构并将值添加到对应的下标位置
    variablesToRemove = Collections.newSetFromMap(new IdentityHashMap<FastThreadLocal<?>, Boolean>());
    threadLocalMap.setIndexedVariable(variablesToRemoveIndex, variablesToRemove);
  } else {
    // 否则直接应用取出的值
    variablesToRemove = (Set<FastThreadLocal<?>>) v;
  }
	// 添加元素到set集合中
  variablesToRemove.add(variable);
}
```

##### 2.get值

```java
public final V get() {
  return get(InternalThreadLocalMap.get());
}

 /**
  * get操作比较简单，直接从threadLocalMap获取对应的下标元素返回。
  */
public final V get(InternalThreadLocalMap threadLocalMap) {
  Object v = threadLocalMap.indexedVariable(index);
  if (v != InternalThreadLocalMap.UNSET) {
    return (V) v;
  }
  return initialize(threadLocalMap);
}

/**
  * 是一个默认值操作，可以通过最后的initValue()会调用FastThreadLocal#initialValue
  * 做一个初始化的操作。
  */
private V initialize(InternalThreadLocalMap threadLocalMap) {
  V v = null;
  try {
    v = initialValue();
  } catch (Exception e) {
    PlatformDependent.throwException(e);
  }

  threadLocalMap.setIndexedVariable(index, v);
  addToVariablesToRemove(threadLocalMap, this);
  return v;
}
```

##### 3.remove

```java
public final void remove() {
  remove(InternalThreadLocalMap.getIfSet());
}

@SuppressWarnings("unchecked")
public final void remove(InternalThreadLocalMap threadLocalMap) {
  if (threadLocalMap == null) {
    return;
  }
	// 删除对应下标值，赋值为UNSET占位符
  Object v = threadLocalMap.removeIndexedVariable(index);
  //删除InternalThreadLocalMap[0]的Set<FastThreadLocal<?>>中的当前FastThreadLocal对象
  removeFromVariablesToRemove(threadLocalMap, this);
  if (v != InternalThreadLocalMap.UNSET) {
    try {
      // 是一个增强，删除之后的业务逻辑，由子类实现，默认是空实现
      onRemoval((V) v);
    } catch (Exception e) {
      PlatformDependent.throwException(e);
    }
  }
}
```

#### 四、总结

​	`FastThreadLocal`实际上采用的是数组的方式进行存储数据，在数据的获取、赋值都是通过下标的方式进行，而`ThreadLocal`是通过`map`结构，先计算哈希值，在进行线性探测的方式进行定位。所以在高并发下，`FastThreadLocal`应该相对高效，但是`FastThread`有一个弊端就是`index`是一直累加，也就是说如果移除了某个变量是通过将对应下标的元素标记为`UNSET`占位，而不进行回收，会无限制增大，会触发扩容等一些问题。



