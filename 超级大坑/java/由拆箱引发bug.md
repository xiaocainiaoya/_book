##### 由拆箱引发bug

```java
public static void main(String[] args) {
  int a = 0;
  Integer b = null;
  System.out.println(a == b); // Exception in thread "main" java.lang.NullPointerException
}
```

由于`b`是包装类型，而`a`是基本数据类型，在比较时对`b`进行拆箱，所以会通过`((Integer)null).intValue()`进行拆箱，从而引发`NPE`。