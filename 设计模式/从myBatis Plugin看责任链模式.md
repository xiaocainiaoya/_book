### 从myBatis Plugin看责任链模式

[TOC]

#### 一、介绍

​	在`mybatis`中从`sql`的解析到最后结果集的返回，经过了一系列的内部组件，比如参数处理器`parameterHandler`，语句处理器`StatementHandler`，结果集处理器`ResultSetHandler`等。若开发者需要对`SQL`执行的某一环节进行一些特定的处理，比如参数类型的转换，数据分页功能，打印执行的`SQL`语句等都可以通过`mybatis`的插件机制实现。

#### 二、mybatis的责任链

​	`mybatis`中就是对内部的一个`List`数组做拦截，业务方通过实现`Interceptor`接口后，将具体的实现类通过`InterceptorChain#addInterceptor`添加到责任链中，当`mybatis`初始化资源时，会调用`InterceptorChain#pluginAll`通过代理的方式，将所有的插件通过逐层代理的方式将内部核心组件(比如`ParameterHandler`)包裹返回一个代理对象。

​	真正执行的地方是由于将内部核心组件都包装成了代理类，所以在调用执行方法时，会被代理对象拦截进入`invoke`方法，根据执行方法所属类以及注解等判断是否执行拦截器或者是执行原方法。

```java
public interface Interceptor {

  /**
   * 拦截执行该方法
   **/
  Object intercept(Invocation invocation) throws Throwable;

  /**
   * 插入
   **/
  Object plugin(Object target);

  /**
   * 设置属性
   **/
  void setProperties(Properties properties);
}

public class InterceptorChain {

  /**
   * 内部就是一个拦截器的List
   */
  private final List<Interceptor> interceptors = new ArrayList<Interceptor>();

  public Object pluginAll(Object target) {
    //循环调用每个Interceptor.plugin方法
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }

  public void addInterceptor(Interceptor interceptor) {
    interceptors.add(interceptor);
  }

  public List<Interceptor> getInterceptors() {
    return Collections.unmodifiableList(interceptors);
  }
}
```

#### 三、过滤器相关责任链

​	在权限校验等一些拦截器中，通常的做法是有多层拦截，比如简单的登录过程，先校验用户名密码是否正确，在校验是否拥有某项操作的操作权限之后才会使得用户获取到资源，但是如果用户名密码校验失败，就没有必要进入第二部的操作权限校验，所以这种场景下使用`mybatis`那种方式的责任链有所不妥。以下是基于在多层拦截下，若某层校验失败，直接拒绝继续往下校验的责任链模式。

```java
/**
 * 拦截器接口
 **/
public interface Filter {

  /**
   * 拦截执行方法
  **/
  Object doFilter(Object target, FilterChain filterChain);
}

/**
 * 具体实现A拦截
 **/
class FilterA implements Filter{

  @Override
  public Object doFilter(Object target, FilterChain filterChain){
    System.out.println("A");
    // 在这里选择是否继续往下走，这种方式会往下走
    return filterChain.doFilter(target);
  }
}

/**
 * 具体实现B拦截
 **/
class FilterB implements Filter{

  @Override
  public Object doFilter(Object target, FilterChain filterChain){
    System.out.println("B");
    // 这种方式会直接返回，不会继续执行其他拦截器(当然了在我的例子中也没有其他拦截器了)
    return "";
  }
}

/**
 * 内部维护一个数组，存储各个拦截器
 **/
public static class FilterChain{

  private List<Filter> filters = new ArrayList<>();

  private Iterator iterator;

  public Object doFilter(Object target){

    if(iterator == null){
      iterator = filters.iterator();
    }

    if(iterator.hasNext()){
      Filter filter = (Filter) iterator.next();
      filter.doFilter(target, this);
    }
    return target;
  }

  public void addFilter(Filter filter){
    filters.add(filter);
  }
}

/**
  * 测试
 **/
public static void main(String[] args) {
  FilterChain filterChain = new FilterChain();
  FilterA filterA = new FilterA();
  FilterB filterB = new FilterB();
  filterChain.addFilter(filterA);
  filterChain.addFilter(filterB);
  filterChain.doFilter("");
}
```

#### 四、总结

​	以上两种责任链的不同形式，其实是应对于不同的业务场景，当需要所有的拦截都走一轮，则采用第一种；当在某个拦截器失败后不继续进行，则采用第二种。在实际的场景中需要综合考虑，采取最符合业务场景的形式进行编码。

