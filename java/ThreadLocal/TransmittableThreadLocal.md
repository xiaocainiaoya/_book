### TransmittableThreadLocal

[TOC]



#### 一、背景

​	上文说到父子线程传递本地变量可以通过`InheritableThreadlocoal`进行传递，但是如果采用线程池，不一定能传递，因为在线程在线程池中的存在不是每次使用都会进行创建，`InheritableThreadlocal`是在线程初始化时`intertableThreadLocals=true`才会进行拷贝传递。所以若本次使用的子线程是已经被池化的线程，从线程池中取出线下进行使用，是没有经过初始化的过程，也就不会进行父子线程的本地变量拷贝。

​	由于在日常应用场景中，绝大多数都是会采用线程池的方式进行资源的有效管理。目前知道的阿里有一个开源项目就是为了解决这个问题`ThansmittableThreadLocal`。

#### 二、简介

​	在使用线程池等会池化复用线程的执行组件情况下，提供`ThreadLocal`值的传递功能，解决异步执行时上下文传递的问题。

​	`JDK`的[`InheritableThreadLocal`](https://docs.oracle.com/javase/10/docs/api/java/lang/InheritableThreadLocal.html)类可以完成父线程到子线程的值传递。但对于使用线程池等会池化复用线程的执行组件的情况，线程由线程池创建好，并且线程是池化起来反复使用的；这时父子线程关系的`ThreadLocal`值传递已经没有意义，应用需要的实际上是把 **任务提交给线程池时**的`ThreadLocal`值传递到 **任务执行时**。

​	本章主要介绍使用线程池场景下的问题，`TransmittableThreadLocal`还有很多其他的应用场景。[传送门](https://github.com/alibaba/transmittable-thread-local)

#### 三、基本使用

> 1.引入`TransimittableThreadLocal`依赖

```xml
<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>transmittable-thread-local</artifactId>
  <version>2.11.5</version>
</dependency>
```

> 2.简单使用

```java
public static void main(String[] args) throws InterruptedException {
  // 创建线程池
  ExecutorService executorService = Executors.newSingleThreadExecutor();
  // 线程池经过TtlExecutors工具类包装，返回包装类ExecutorServiceTtlWrapper
  executorService = TtlExecutors.getTtlExecutorService(executorService);
	// 创建需要传递给线程池本地变量
  TransmittableThreadLocal<String> username = new TransmittableThreadLocal<>();
	
  // 首次调用,这时候线程池中还未有线程，就算不使用TTL也可以通过InheritableThreadLocal获取到
  // 父线程的本地变量。
  username.set("zhangShang");
  executorService.submit(new Runnable() {
    @Override
    public void run() {
      System.out.println(username.get());
    }
  });
	
  // 第二次调用，由于使用的是单一线程的线程池，这时候是复用了上面创建的线程，所以这时通过
  // inheritableThreadLocal是获取不到本地变量的。
  Thread.sleep(3000);
  username.set("liSi");
  executorService.submit(new Runnable() {
    @Override
    public void run() {
      System.out.println(username.get());
    }
  });

  Thread.sleep(3000);
  username.set("wangWu");
  executorService.submit(new Runnable() {
    @Override
    public void run() {
      System.out.println(username.get());
    }
  });
}
```

输出结果为:

```java
zhangShang
liSi
wangWu
```

#### 四、原理

> 从定义来看，`TransimittableThreadLocal`继承于`InheritableThreadLocal`，并实现`TtlCopier`接口，它里面只有一个`copy`方法。所以主要是对`InheritableThreadLocal`的扩展。

```java
public class TransmittableThreadLocal<T> extends InheritableThreadLocal<T> implements TtlCopier<T> 
```

> 在`TransimittableThreadLocal`中添加`holder`属性。这个属性的作用就是被标记为具备线程传递资格的对象都会被添加到这个对象中。**要标记一个类，比较容易想到的方式，就是给这个类新增一个`Type`字段，还有一个方法就是将具备这种类型的的对象都添加到一个静态全局集合中。之后使用时，这个集合里的所有值都具备这个标记。**

```java
// 1. holder本身是一个InheritableThreadLocal对象
// 2. 这个holder对象的value是WeakHashMap<TransmittableThreadLocal<Object>, ?>
// 		2.1 WeekHashMap的value总是null,且不可能被使用。
//    2.2 WeekHasshMap支持value=null
private static InheritableThreadLocal<WeakHashMap<TransmittableThreadLocal<Object>, ?>> holder = new InheritableThreadLocal<WeakHashMap<TransmittableThreadLocal<Object>, ?>>() {
  @Override
  protected WeakHashMap<TransmittableThreadLocal<Object>, ?> initialValue() {
    return new WeakHashMap<TransmittableThreadLocal<Object>, Object>();
  }
	
  /**
   * 重写了childValue方法，实现上直接将父线程的属性作为子线程的本地变量对象。
   */
  @Override
  protected WeakHashMap<TransmittableThreadLocal<Object>, ?> childValue(WeakHashMap<TransmittableThreadLocal<Object>, ?> parentValue) {
    return new WeakHashMap<TransmittableThreadLocal<Object>, Object>(parentValue);
  }
};
```

> 应用代码是通过`TtlExecutors`工具类对线程池对象进行包装。工具类只是简单的判断，输入的线程池是否已经被包装过、非空校验等，然后返回包装类`ExecutorServiceTtlWrapper`。根据不同的线程池类型，有不同和的包装类。

```java
@Nullable
public static ExecutorService getTtlExecutorService(@Nullable ExecutorService executorService) {
  if (TtlAgent.isTtlAgentLoaded() || executorService == null || executorService instanceof TtlEnhanced) {
    return executorService;
  }
  return new ExecutorServiceTtlWrapper(executorService);
}
```

> 进入包装类`ExecutorServiceTtlWrapper`。可以注意到不论是通过`ExecutorServiceTtlWrapper#submit`方法或者是`ExecutorTtlWrapper#execute`方法，都会将线程对象包装成`TtlCallable`或者`TtlRunnable`，用于在真正执行`run`方法前做一些业务逻辑。

```java
/**
 * 在ExecutorServiceTtlWrapper实现submit方法
 */
@NonNull
@Override
public <T> Future<T> submit(@NonNull Callable<T> task) {
  return executorService.submit(TtlCallable.get(task));
}

/**
 * 在ExecutorTtlWrapper实现execute方法
 */
@Override
public void execute(@NonNull Runnable command) {
  executor.execute(TtlRunnable.get(command));
}

```

> 所以，重点的核心逻辑应该是在`TtlCallable#call()`或者`TtlRunnable#run()`中。以下以`TtlCallable`为例，`TtlRunnable`同理类似。在分析`call()`方法之前，先看一个类`Transmitter`

```java
public static class Transmitter {
  /**
    * 捕获当前线程中的是所有TransimittableThreadLocal和注册ThreadLocal的值。
    */
  @NonNull
  public static Object capture() {
    return new Snapshot(captureTtlValues(), captureThreadLocalValues());
  }
	
    /**
    * 捕获TransimittableThreadLocal的值,将holder中的所有值都添加到HashMap后返回。
    */
  private static HashMap<TransmittableThreadLocal<Object>, Object> captureTtlValues() {
    HashMap<TransmittableThreadLocal<Object>, Object> ttl2Value = 
      new HashMap<TransmittableThreadLocal<Object>, Object>();
    for (TransmittableThreadLocal<Object> threadLocal : holder.get().keySet()) {
      ttl2Value.put(threadLocal, threadLocal.copyValue());
    }
    return ttl2Value;
  }

  /**
    * 捕获注册的ThreadLocal的值,也就是原本线程中的ThreadLocal,可以注册到TTL中，在
    * 进行线程池本地变量传递时也会被传递。
    */
  private static HashMap<ThreadLocal<Object>, Object> captureThreadLocalValues() {
    final HashMap<ThreadLocal<Object>, Object> threadLocal2Value = 
      new HashMap<ThreadLocal<Object>, Object>();
    for(Map.Entry<ThreadLocal<Object>,TtlCopier<Object>>entry:threadLocalHolder.entrySet()){
      final ThreadLocal<Object> threadLocal = entry.getKey();
      final TtlCopier<Object> copier = entry.getValue();
      threadLocal2Value.put(threadLocal, copier.copy(threadLocal.get()));
    }
    return threadLocal2Value;
  }

  /**
    * 将捕获到的本地变量进行替换子线程的本地变量，并且返回子线程现有的本地变量副本backup。
    * 用于在执行run/call方法之后，将本地变量副本恢复。
    */
  @NonNull
  public static Object replay(@NonNull Object captured) {
    final Snapshot capturedSnapshot = (Snapshot) captured;
    return new Snapshot(replayTtlValues(capturedSnapshot.ttl2Value), 
                        replayThreadLocalValues(capturedSnapshot.threadLocal2Value));
  }
	
  /**
    * 替换TransmittableThreadLocal
    */
  @NonNull
  private static HashMap<TransmittableThreadLocal<Object>, Object> replayTtlValues(@NonNull HashMap<TransmittableThreadLocal<Object>, Object> captured) {
    // 创建副本backup
    HashMap<TransmittableThreadLocal<Object>, Object> backup = 
      new HashMap<TransmittableThreadLocal<Object>, Object>();

    for (final Iterator<TransmittableThreadLocal<Object>> iterator = holder.get().keySet().iterator(); iterator.hasNext(); ) {
      TransmittableThreadLocal<Object> threadLocal = iterator.next();
      // 对当前线程的本地变量进行副本拷贝
      backup.put(threadLocal, threadLocal.get());

      // 若出现调用线程中不存在某个线程变量，而线程池中线程有，则删除线程池中对应的本地变量
      if (!captured.containsKey(threadLocal)) {
        iterator.remove();
        threadLocal.superRemove();
      }
    }
    // 将捕获的TTL值打入线程池获取到的线程TTL中。
    setTtlValuesTo(captured);
    // 是一个扩展点，调用TTL的beforeExecute方法。默认实现为空
    doExecuteCallback(true);
    return backup;
  }

  private static HashMap<ThreadLocal<Object>, Object> replayThreadLocalValues(@NonNull HashMap<ThreadLocal<Object>, Object> captured) {
    final HashMap<ThreadLocal<Object>, Object> backup = 
      new HashMap<ThreadLocal<Object>, Object>();
    for (Map.Entry<ThreadLocal<Object>, Object> entry : captured.entrySet()) {
      final ThreadLocal<Object> threadLocal = entry.getKey();
      backup.put(threadLocal, threadLocal.get());
      final Object value = entry.getValue();
      if (value == threadLocalClearMark) threadLocal.remove();
      else threadLocal.set(value);
    }
    return backup;
  }

  /**
    * 清除单线线程的所有TTL和TL，并返回清除之气的backup
    */
  @NonNull
  public static Object clear() {
    final HashMap<TransmittableThreadLocal<Object>, Object> ttl2Value = 
      new HashMap<TransmittableThreadLocal<Object>, Object>();

    final HashMap<ThreadLocal<Object>, Object> threadLocal2Value = 
      new HashMap<ThreadLocal<Object>, Object>();
    for(Map.Entry<ThreadLocal<Object>,TtlCopier<Object>>entry:threadLocalHolder.entrySet()){
      final ThreadLocal<Object> threadLocal = entry.getKey();
      threadLocal2Value.put(threadLocal, threadLocalClearMark);
    }
    return replay(new Snapshot(ttl2Value, threadLocal2Value));
  }

  /**
    * 还原
    */
  public static void restore(@NonNull Object backup) {
    final Snapshot backupSnapshot = (Snapshot) backup;
    restoreTtlValues(backupSnapshot.ttl2Value);
    restoreThreadLocalValues(backupSnapshot.threadLocal2Value);
  }

  private static void restoreTtlValues(@NonNull HashMap<TransmittableThreadLocal<Object>, Object> backup) {
    // 扩展点，调用TTL的afterExecute
    doExecuteCallback(false);

    for (final Iterator<TransmittableThreadLocal<Object>> iterator = holder.get().keySet().iterator(); iterator.hasNext(); ) {
      TransmittableThreadLocal<Object> threadLocal = iterator.next();

      if (!backup.containsKey(threadLocal)) {
        iterator.remove();
        threadLocal.superRemove();
      }
    }

    // 将本地变量恢复成备份版本
    setTtlValuesTo(backup);
  }

  private static void setTtlValuesTo(@NonNull HashMap<TransmittableThreadLocal<Object>, Object> ttlValues) {
    for (Map.Entry<TransmittableThreadLocal<Object>, Object> entry : ttlValues.entrySet()) {
      TransmittableThreadLocal<Object> threadLocal = entry.getKey();
      threadLocal.set(entry.getValue());
    }
  }

  private static void restoreThreadLocalValues(@NonNull HashMap<ThreadLocal<Object>, Object> backup) {
    for (Map.Entry<ThreadLocal<Object>, Object> entry : backup.entrySet()) {
      final ThreadLocal<Object> threadLocal = entry.getKey();
      threadLocal.set(entry.getValue());
    }
  }

  /**
   * 快照类，保存TTL和TL
   */
  private static class Snapshot {
    final HashMap<TransmittableThreadLocal<Object>, Object> ttl2Value;
    final HashMap<ThreadLocal<Object>, Object> threadLocal2Value;

    private Snapshot(HashMap<TransmittableThreadLocal<Object>, Object> ttl2Value,
                     HashMap<ThreadLocal<Object>, Object> threadLocal2Value) {
      this.ttl2Value = ttl2Value;
      this.threadLocal2Value = threadLocal2Value;
    }
  }
```

> 进入`TtlCallable#call()`方法。

```java
@Override
public V call() throws Exception {
  Object captured = capturedRef.get();
  if (captured == null || releaseTtlValueReferenceAfterCall && 
      !capturedRef.compareAndSet(captured, null)) {
    throw new IllegalStateException("TTL value reference is released after call!");
  }
  // 调用replay方法将捕获到的当前线程的本地变量，传递给线程池线程的本地变量，
  // 并且获取到线程池线程覆盖之前的本地变量副本。
  Object backup = replay(captured);
  try {
    // 线程方法调用
    return callable.call();
  } finally {
    // 使用副本进行恢复。
    restore(backup);
  }
}
```

​	到这基本上线程池方式传递本地变量的核心代码已经大概看完了。总的来说在创建`TtlCallable`对象是，调用`capture()`方法捕获调用方的本地线程变量，在`call()`执行时，将捕获到的线程变量，替换到线程池所对应获取到的线程的本地变量中，并且在执行完成之后，将其本地变量恢复到调用之前。







