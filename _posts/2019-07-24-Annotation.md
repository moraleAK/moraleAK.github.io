---
title: Annotation
layout: posts
categories: Tree
---

# Annotation
    用到的地方可以得到不同的 类/方法 中注解的各种参数与值
    注解也就是Annotation,从JDK5开始，java增加了对元数据（描述数据属性的信息）的支持。
    其实说白就是代码里的特殊标志，这些标志可以在编译，类加载，运行时被读取，并执行相应的处理，以便于其他工具补充信息或者进行部署。
    注解本身没有任何作用，仅仅只是一个标识，标识里可以设定一些属性例如：name, value 等，注解的实现是在需要的地方通过反射获取对应的注解类型
    同时获取对应类型里的值，相关实现根据获取到的值实现相应的逻辑代码
    
## 获取注解
    一般是通过反射的方式获取方法、类或者是属性上的注解
### 示例代码
{% highlight java linenos %} 
import org.junit.Test;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author Gin
 * @since 2019/7/24 00:04
 */
@RestController(value = "test")
public class Annotation {

    @Test
    public void test(){
        RestController restController = Annotation.class.getAnnotation(RestController.class);
        System.out.println(restController.value());
    }
}
{% endhighlight %}
### 输出结果
> test

## 组合注解
### 上文 @RestController
    @RestController 注解又被@Controller、@ResponseBody注解
{% highlight java linenos %} 

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import org.springframework.core.annotation.AliasFor;
import org.springframework.stereotype.Controller;

/**
 * A convenience annotation that is itself annotated with
 * {@link Controller @Controller} and {@link ResponseBody @ResponseBody}.
 * <p>
 * Types that carry this annotation are treated as controllers where
 * {@link RequestMapping @RequestMapping} methods assume
 * {@link ResponseBody @ResponseBody} semantics by default.
 *
 * <p><b>NOTE:</b> {@code @RestController} is processed if an appropriate
 * {@code HandlerMapping}-{@code HandlerAdapter} pair is configured such as the
 * {@code RequestMappingHandlerMapping}-{@code RequestMappingHandlerAdapter}
 * pair which are the default in the MVC Java config and the MVC namespace.
 *
 * @author Rossen Stoyanchev
 * @author Sam Brannen
 * @since 4.0
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller
@ResponseBody
public @interface RestController {

	/**
	 * The value may indicate a suggestion for a logical component name,
	 * to be turned into a Spring bean in case of an autodetected component.
	 * @return the suggested component name, if any (or empty String otherwise)
	 * @since 4.0.1
	 */
	@AliasFor(annotation = Controller.class)
	String value() default "";

}
{% endhighlight %}

### 获取注解上的注解
    如同获取类、方法、属性上的注解一样，通过反射拿到对应的注解
{% highlight java linenos %} 
import org.junit.Test;
import org.springframework.web.bind.annotation.RestController;

import java.util.Arrays;

/**
 * @author Gin
 * @since 2019/7/24 00:04
 */
@RestController(value = "test")
public class Annotation {

    @Test
    public void test(){
        Arrays.asList(RestController.class.getAnnotations()).forEach(System.out::println);
    }
}

{% endhighlight %}
---
    输出结果
    
> @java.lang.annotation.Target(value=[TYPE])    
> @java.lang.annotation.Retention(value=RUNTIME)    
> @java.lang.annotation.Documented()    
> @org.springframework.stereotype.Controller(value=)    
> @org.springframework.web.bind.annotation.ResponseBody()   

### 生效
    组合注解的生效实际就是实现的逻辑部分是否循环获取对应属性上的注解，如果仅仅直接获取注解，那注解注解的注解是不会被获取到的
    
    直接获取注解
{% highlight java linenos %} 
@RestController(value = "test")
public class Annotation {

    @Test
    public void test(){
        Arrays.asList(Annotation.class.getAnnotations()).forEach(System.out::println);
    }
}
{% endhighlight %}
    循环获取,此处未采取递归方式，仅获取深度为2的注解
{% highlight java linenos %} 
import org.junit.Test;
import org.springframework.web.bind.annotation.RestController;

import java.util.Arrays;

/**
 * @author Gin
 * @since 2019/7/24 00:04
 */
@RestController(value = "test")
public class Annotation {

    @Test
    public void test(){
        Arrays.asList(Annotation.class.getAnnotations())
                .forEach(e -> Arrays.asList(e.annotationType().getAnnotations())
                        .forEach(System.out::println));
    }
}
{% endhighlight %}
---
    输出结果
    
> @java.lang.annotation.Target(value=[TYPE])    
> @java.lang.annotation.Retention(value=RUNTIME)    
> @java.lang.annotation.Documented()    
> @org.springframework.stereotype.Controller(value=)    
> @org.springframework.web.bind.annotation.ResponseBody()   

## Spring 组合注解生效以及自定义注解生效
    Spring 会递归查找该方法上注解以及注解上的注解
