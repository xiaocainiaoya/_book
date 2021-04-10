### java的泛型

[TOC]

#### 一、介绍

​	泛型实现了参数化参数化类型的概念，是代码可以应用于多种类型，设计的初衷应该是希望类或者方法能够具备最广泛的表达能力。在引入泛型之前，一般都是依赖于`Object`顶层对象实现类似泛型的功能，但是使用`Object`有一个缺陷是如果类型转换异常，编译器在编译期无法检测这种异常，只有在字节码的运行时期才会抛出类型转换异常。而`JDK 1.5`之后引入的泛型，在编译期就会对类型进行检查，使得问题可以及早发现。

#### 二、泛型方式

##### 1. 泛型类

> 泛型类的写法是在类上指明参数，并在属性或者方法中使用。

先来看一下在没有泛型之前的操作方式，采用`Object`的方式，使得调用点获取到这个对象，如果需要获取到对象自身实现的某个方法，就需要进行强制类型装换，所以可能会出现类型装换异常。

```java
public static class Animal {

  private Object animal;

  public void set(Object animal){
    this.animal = animal;
  }
  public Object get(){
    return animal;
  }
}
```

再泛型引入之后采用如下方式，调用点在通过`get`方法获取到对象时，直接是调用点设置的参数类型，所以无需进行强制类型转换。

```java
public static class Animal<T> {

  private T animal;

  public void set(T animal){
    this.animal = animal;
  }
  public T get(){
    return animal;
  }
}
```

##### 2.泛型接口

> 泛型也可以应用于接口，使用上与泛型类类似。

泛型接口对调用点来说有两种方式，一种是实现类指定了参数类型，则调用点无需再参数化类型；另一种是实现类依旧采用泛型的方式继承，则调用点就需要参数化类型。

```java
public interface People<T> {
    public T display();
}

class Girl implements People<String>{
	public void display(String str){
		System.out.println("display: "+ str);
	}
}

class Boy<T> implements People<T>{
	public void display(T t){
		System.out.println("display:"+t);
	}
}
```

##### 3.泛型方法

> 泛型也可以应用于方法上，并且这个方法对应的类可以是泛型类也可以不是泛型类。定义泛型方法只需要将泛型列表置于返回值之前。

```java
class Test{
	public <T>  void display(T t){
		System.out.println(t);
	}
}
```

**显式的类型说明：**对泛型方法的调用，可以显式的指明类型，语法是在点操作符与方法名之间插入尖括号，然后将类型置于尖括号中，这种方式广泛用于静态方法，使用`tk.mybatis`过都知道`Example`中创建的静态泛型方法，笔者刚开始进入公司写的时候都是按部就班，不知道泛型方法的具体逻辑，学了泛型才知道。

```java
// ========= WeekendSqls ============
public class WeekendSqls<T> implements SqlsCriteria {
    public static <T> WeekendSqls<T> custom() {
        return new WeekendSqls();
    }
}

// 调用点
public static void main(String[] args) {
	WeekendSqls<People> weekendSql = WeekendSqls.<People>custom();
}
```

#### 三、泛型擦除

> java的泛型是使用擦除来实现的，所以在真正运行的时候，任何具体的类型都会被擦除，唯一知道的是，在使用某一对象，所以对以下例子而言，最后的输出结果为`true`，类型都为`java.util.ArrayList`

```java
public static void main(String[] args) {
  Class c1 = new ArrayList<String>().getClass();
  Class c2 = new ArrayList<Integer>().getClass();
  System.out.println("c1的类型为:" + c1 + "\n" + "c2的类型为" + c2);
  System.out.println( c1 == c2);
}
```

##### 1. 泛型数组

由于`Class<T>`在运行时已经被擦除，实际的结果为`Class`，而通过这个没有指定类型`Class`的`newInstance`方法不会产生具体的结果。

```java
public class ArrayMaker<T> {

    private Class<T> kind;

    public ArrayMaker(Class<T> kind){
      this.kind = kind;
    }

    T[] create(int size){
      return (T[]) Array.newInstance(kind, size);
    }

    public static void main(String[] args) {
      ArrayMaker<String> stringArrayMaker =  new ArrayMaker<>(String.class);
      String[] stringArray = stringArrayMaker.create(9);
      // 这里的输出结果为 [null, null, null, null, null, null, null, null, null]  
      System.out.println(Arrays.toString(stringArray));
    }
}
```

再看一个例子

```java
public class ArrayOfGeneric {

    public static class Generic<T>{}

    static final int SIZE = 100;
		// 在编译时期是Generic<Integer>[]，在运行时期可以理解为Object[]
    static Generic<Integer>[] gia;

    public static void main(String[] args) {
      // 编译时只会报警告，运行时会抛出类型装换异常，因为从Object[]转Generic<Integer>[]
      // gia = (Generic<Integer>[]) new Object[SIZE];
      // 由于是Generic<Integer>[]强转Object[]，所以运行正常。
      gia = (Generic<Integer>[]) new Generic[SIZE];
    }
}
```

再看一个更复杂一点的例子

```java
public class GenericArray<T> {

    private T[] array;

    private GenericArray(int size){
      // 从编译器层面来说，创建数组强制装换为T[]，只会报警告
      // 从运行层面来说，Object[]强制装换为Object[]，正常行为
      array = (T[]) new Object[size];
    }

    public void put(int index, T item){
      array[index] = item;
    }

    public T[] rep(){
      return array;
    }

    public static void main(String[] args) {
      GenericArray<Integer> gai = new GenericArray<>(10);
      // 从编译器层面来说，获取到T[]赋值为Object[],正常行为
      // 从运行层面来说，Object[]赋值为Object[]，正常行为
      Object[] oa = gai.rep();
    }
}
```

通过以上的例子，在实际的编写代码中要考虑到编译器层面和运行层面，对编译器来说，需要保证类型的异常转换都在编译时期通过警告或者编译不通过的方式提示用户；对运行层面来说，由于参数化类型已经被擦除，有可能会导致出现类型转换异常。总之一句话，编译器在编译时就是尽可能的做类型检查，前置了异常的抛出时机，避免所有的类型转换异常都在运行时期抛出。

##### 2.边界

**2.1上界**

> 泛型上界采用`<? extends T>`表示当前泛型参数只能由`T`类型的子类构成。

`<? extends T>`指明了这个泛型类参数化类型的参数的只能是`T`的子类，且会影响到泛型类中**入参**为参数化类型的方法。

```java
class Food{}
class Fruit extend Food{}
class Apple extends Fruit {}
class Orange extends Fruit{}

class Plate<T>{
  private T item;
  public Plate(T t){item = t;}
  public void set(T t){item = t;}
  public T get(){return item;}
}

public static void main(String[] args) {
  Plate<? extends Fruit> plate = new Plate<>(new Apple());
  // 两个set方法均报错，由于限定了参数化类型的上界，而对于Fruit来说有很多子类
  // 编译器在这时不知道应该使用哪个类来创建引用。
  plate.set(new Apple());
  plate.set(new Fruit());
  
  Fruit f = plate.get();
  // 报错，只能通过上界类获取引用
  Apple a = plate.get();
}
```

**2.2下界**

> 泛型下界采用`<? super T>`表示当前泛型的参数只能有`T`类型的父类构成。

`<? super T>`指明了这个泛型类参数化类型的参数的只能是`T`的父类，且会影响到泛型类中**返回值**为参数化类型的方法。还是上面的例子

```java
public static void main(String[] args) {
  Plate<? super Fruit> pf = new Plate<>(new Fruit());
  // 由于限定参数类型为Fruit的超类，所以添加的元素只要是Fruit以及Fruit的
  // 子类都会成功
  pf.set(new Apple());
  pf.set(new Fruit());
	
  // 报错，由于限定参数为Fruit的超类，不能用Fruit来引用，当然了就算是Food也不行
  Fruit ff = pf.get();
}
```

**2.3无界(`?`通配符)**

> `?`称为无界通配符，表示的是一种未知类型，所以一般如果采用了`?`定义一个泛型，对其调用的是与具体类型无关的方法。最常用的应该是`Class<?>`，因为就算是使用泛型`Class<T>`也并没有依赖于`T`

如果看过`jdk`容器相关的源码，都应该知道在容器中有很多的方法都采用这种写法，即无需关心具体的类型。

```java
public boolean containsAll(Collection<?> c) { 
  return c.isEmpty(); 
}
```

`?`表示的未知类型，相比于`Object`应该来说是一个更大的概念，所以`List<?> != List<Object>`，并且`List<Object>`不能指向`List<?>`的引用；而`List<?>`可以指向`List<Object>`的引用。

但是有一点需要注意若`List<?>`指向`List<Object>`之后，由于类型是未知的，所以`List`中使用泛型的方法都不能使用，也就是`add(E e)`不能使用，编译器报错；而`remove(Object o)`参数没有使用泛型，则可以使用。

```java
List<?> list = new ArrayList<>();
List<Object> objects = list;

List<Object> objects = new ArrayList<>();
List<?> list = objects;
```

**2.4 小结**

​	不论使用哪种边界，对于存在`?`来说，表示的都是未知类型，所以在使用上下界处理时要精准的知道类型之间的继承关系，上下界对**入参**参数化类型和**返回值**参数类型行为上的区别，并且在合适的场景可以进行添加操作，合适的场景可以进行获取操作。根据`PECS(Producer Extends Consumer Super)`原则，频繁读取操作，适合使用上界`extends`，频繁插入操作，适合使用下界`Super`。

**2.5 不使用通过符`?`的上下界**

> 形如<T extends Fruit>或者是<T Super Fruit>，这种方式在声明处就指定了参数化类型的值。

修改`Plate`类

```java
static class Plate<T extends Fruit>{
  private T item;
  public Plate(T t){item = t;}
  public void set(T t){item = t;}
  public T get(){return item;}
}
public static void main(String[] args) {
	// 在声明处指定参数化类型的值
  Plate<Fruit> pf = new Plate<>(new Apple());
  pf.set(new Apple());
  pf.set(new Fruit());
	
  // 此处声明参数化类型的值为Apple
  Plate<Apple> pa = new Plate<>(new Apple());
  pa.set(new Apple());
  // 编译报错，指定只能传入Apple对象
  pa.set(new Fruit());
}
```

#### 四、常见问题

##### 1. 基本数据类型不能作为类型参数

​	在泛型中不能使用基本数据类型作为类型的参数，也就是不允许`ArrayList<int>`的方式，只能通过`java`的自动装箱拆箱机制，使用`ArrayList<Integer>`来实现。

##### 2.重载问题

​	当出现多个参数化类型时，由于类型擦除的原因，重载的方法实际产生的是一样类型签名，所以不能产生不同类型的参数列表，必须提供明显有区别的方法名。

```java
    static class Plate<T, K>{
        private T item;
        private K key;
        public Plate(T t){item = t;}
       // 编译报错
        public void set(T t){item = t;}
        public void set(K k){key = k;}
        public T get(){return item;}
    }
```

##### 3. 自限定的泛型

自限定泛型强调的是创建这个类所使用的参数与这个类具有相同的类型。感觉有点绕，下面看一下`java`编程思想中的例子。

```java
// 采用自限定声明
class SelfBounded<T extends SelfBounded<T>> { 
    T element;
    SelfBounded<T> set(T arg) {
     element = arg;
     return this;
    }
    T get() { return element; }
}

/*
 * 自限定类型的使用就两种方式，就是以下两种方式。
 * 1. 这边为了引入概念来说明，标记class之后的A为A1，尖括号中的A为A2
 *    创建的这个类A1所使用的参数A2与这个类A1具有相同的类型。
 */
class A extends SelfBounded<A> {}
/*
 * 2. 由于A已经继承了SelfBounded<A>，所以B可以直接继承
 */
class B extends SelfBounded<A> {} // It's OK.

class C extends SelfBounded<C> {
    C setAndGet(C arg) { set(arg); return get(); }
}

class D {}
// class E extends SelfBounded<D> {} // [Compile error]: Type parameter D is not within its bound
        
public class SelfBounding {
    public static void main(String[] args) {
        A a = new A();
        a.set(new A());
        a = a.set(new A()).get();
        a = a.get();
        C c = new C();
        c = c.setAndGet(new C());
    }
}
```

`Enum`的设计正是采用泛型自限定的方式。`Enum`的泛型限定了`E`的上界为`Enum`自身，确保了`Enum`的子类才能作为泛型参数，而在枚举中`compareTo(E o)`，在比较时，希望的是传入的参数类型就是`Enum`类型。所以这种设计使得方法中传入参数和返回的方法是与创建的类型保持继承关系，也就是说`E extends Enum<E>`保证`Enum<E>`的子类，比如`StatusEnum`枚举类都能够接收或者返回其本身。

```java
public abstract class Enum<E extends Enum<E>> implements Comparable<E>, Serializable
```















