

#### Lombok的@SneakyThrows详解

[TOC]

#### 一、简介

​	在`java`的异常体系中`Exception`异常有两个分支，一个是运行时异常`RuntimeException`，一个是编译时异常，在`Exception`下的所有非`RuntimeException`异常，比如`IOException`、`SQLException`等；所有的运行时异常不捕获，编译时异常是一定要捕获，否则编译会报错。`@SneakyThrows`就是利用了这一机制，将当前方法抛出的异常，包装成`RuntimeException`，骗过编译器，使得调用点可以不用显示处理异常信息。

#### 二、原理

```java
/*
 * 若不使用@SneakyThrows注解，newInsstance方法会要求抛出InstantiationException, 
 * IllegalAccessException异常，且调用sneakyThrowsTest()的地方需要捕获这些异常，
 * 加上@SneakyThrows注解之后就不需要捕获异常信息。
 */
@SneakyThrows
private void sneakyThrowsTest(){
  SneakyThrowsDemo.class.newInstance();
}
```

**如下为反编译之后的结果**

```java
private void sneakyThrowsTest() {
    try {
      HelloController.class.newInstance();
    } catch (Throwable e) {
      // 调用Lombok方法转化为RuntimeException
      throw Lombok.sneakyThrow(e);
    }
}


// =========== ombok =========
public static RuntimeException sneakyThrow(Throwable t) {
  if (t == null) {
    throw new NullPointerException("t");
  } else {
    return Lombok.<RuntimeException>sneakyThrow0(t);
  }
}

/*
 * 这个方法是关键，这里对入参类型的约束为<T extends Throwable>，而调用点将异常强转为
 * RuntimeException
 */
private static <T extends Throwable> T sneakyThrow0(Throwable t) throws T {
  throw (T)t;
}
```

那么问题来了，为什么这个地方可以对原来的异常进行强转为`RuntimeExcption`？以下为直接强转的代码，显然运行之后报类型转换异常。

```java
private void sneakyThrowsTest() {
  try {
    throw new Exception();
  } catch (Throwable e) {
		// 直接将e强转为RuntimeException，运行到这里会报类型转换异常。
    throw (RuntimeException)e;
  }
}
```

实际上，这种做法是一种通过泛型欺骗了编译器，让编译器在编译期不报错(报警告)，而最后在`JVM`虚拟机中执行的字节码的并没有区别编译时异常和运行时异常，只有是不是和抛不抛异常而已。































