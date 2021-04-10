### SpringMVC中的过滤器和拦截器

[TOC]

#### 一、过滤器

> ​	过滤器`Filter`是通过实现`java.servlet.filter`接口实现过滤器功能，作用是用于对传入的`request`和响应的`response`进行一些处理，比如对请求参数进行校验，或者设置、检验头部信息，再或者对一些非法行为进行校验。由实现的接口可知，过滤器是依赖于`servlet`容器。**所以由于过滤器不依赖于`spring`容器，它也就无法获取到容器中的对象。**

创建一个过滤器类继承`java.servlet.filter`接口，实现`filter`中的拦截方法。

```java
package com.example.demo.modules.filter;

import javax.servlet.*;
import java.io.IOException;

/**
 * @Description:
 * @date :  2020/6/9
 */
public class MyFilter implements Filter {

	@Override
	public void init(FilterConfig filterConfig) throws ServletException {

	}

	@Override
	public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
		// 过滤器的具体执行方法
	}

	@Override
	public void destroy() {

	}
}
```

2. 把创建的过滤器类加入过滤器链中

```java
package com.example.demo.modules.filter;

import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.servlet.Filter;

/**
 * 创建两个过滤器myFilter和myFilter2，并且分别添加到FilterRegistrationBean
 * @Description:
 * @date :  2020/6/9
 */
@Configuration
public class ConfigBean {

	@Bean
	public FilterRegistrationBean filterRegistrationBean(){
		FilterRegistrationBean filterRegistrationBean = new 
            											FilterRegistrationBean();
		filterRegistrationBean.setFilter(myFilter());
		filterRegistrationBean.setName("myFilter");
		filterRegistrationBean.addUrlPatterns("/*");
		filterRegistrationBean.setOrder(1);
		//filterRegistrationBean.setInitParameters();
		return filterRegistrationBean;
	}

	@Bean
	public FilterRegistrationBean filter2RegistrationBean(){
		FilterRegistrationBean filterRegistrationBean = new
            										FilterRegistrationBean();
		filterRegistrationBean.setFilter(myFilter2());
		filterRegistrationBean.setName("myFilter2");
		filterRegistrationBean.addUrlPatterns("/*");
		filterRegistrationBean.setOrder(2);
		//filterRegistrationBean.setInitParameters();
		return filterRegistrationBean;
	}

	@Bean
	public Filter myFilter(){
		return new MyFilter();
	}

	@Bean
	public Filter myFilter2(){
		return new MyFilter();
	}
}
```

#### 二、拦截器

> 拦截器`Interceptor`是通过实现`org.springframework.web.servlet`包的`HandlerInterceptor`接口实现，这个接口是`spring`容器的接口，所以它是依赖于`spring`容器的。主要作用是`AOP`的思想，可以对某一个方法进行横切，做一些业务逻辑。

**1.编写自定义拦截器类**

```java
package com.example.demo.modules.interceptor;

import lombok.extern.slf4j.Slf4j;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

/**
 * @Description:
 * @date :  2020/6/9
 */
@Slf4j
public class CustomHandlerInterceptor implements HandlerInterceptor {

	@Override
	public boolean preHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o) throws Exception {
		log.info("请求调用之前");
		return false;
	}

	@Override
	public void postHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, ModelAndView modelAndView) throws Exception {
		log.info("请求调用之后");
	}

	@Override
	public void afterCompletion(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, Exception e) throws Exception {
		log.info("afterCompletion:请求调用完成后回调方法，即在视图渲染完成后回调");
	}
}

```

**2.注册自定义拦截器**

```java
package com.example.demo.modules.interceptor;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;

/**
 * 在spring2.0中WebMvcConfigurerAdapter已经过时，这里只是为了演示效果，
 * 有兴趣可以看下spring2.0中的WebMvcConfigurer
 * @Description:
 * @date :  2020/6/9
 */
@Configuration
public class WebMvcConfigurer extends WebMvcConfigurerAdapter {

	@Override
	public void addInterceptors(InterceptorRegistry registry) {
		//super.addInterceptors(registry);
		registry.addInterceptor(getHandlerInterceptor()).addPathPatterns("/*");
	}

	@Bean
	public static HandlerInterceptor getHandlerInterceptor(){
		return new CustomHandlerInterceptor();
	}
}

```

**过滤器和拦截器执行过程图**

​	在请求到达容器前，进入`Filter`过滤器链，执行完过滤器链上每个`Filter#doFilter()`方法后，进入`Servlet#service()`方法，然后由`dispatcher`分发器将请求方法给对应映射成功的处理器`controller`，在进入`controller`具体方法之前，会被先进入`Interceptor#preHandler()`方法，然后再进入`controller`的具体返回，执行之后进入`Interceptor#postHandler()`这里主要是拦截了`controller`方法执行之后到返回的数据模型到达视图解析器之前，接着进入`Interceptor#afterCompletion()`方法，主要可以操作返回客户端之前的逻辑，最后返回到过滤链中各个`Filter`的调用点，可以处理返回到客户端的跳转等逻辑。

![](https://gitee.com/jiangjiamin/image-bed/raw/master/upic/2020-11/GvwS1P.png)



#### 三、小结

​	过滤器是`servlet`中的接口，主要可以用于在请求进入到`servlet`之前拦截请求`HttpServletRequest`并根据需要进行一些检查等逻辑操作，也可以在`HttpServletResponse`返回到客户端之前进行一些逻辑操作。

​	拦截器是`spring`中的接口，所以它可以获取到`spring`中的一些`bean`和其他的一些资源，在面向切面编程中应用比较广，拦截其实就是一种`AOP`策略。

























