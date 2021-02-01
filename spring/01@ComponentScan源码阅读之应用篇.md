# @ComponentScan源码阅读之应用篇



> 上一篇说了注解版的启动流程，也就是走个过场，没仔细讲，让大家知道了Spring在启动的时候都启动了哪些类，大概做了什么事。心里有个底，顶多也就算个暖场。从这篇起，Spring的源码阅读就正视开始，难度也会随之增加。

遵循我的原则，我先将`@ComponentScan`的应用



## @ComponentScan的应用

要说应用还得看人家Spring给我们提供了什么样的API。所以，我先看一下`@ComponentScan`内部长啥样

刚刚看了一下代码，实在太多了。不管是截图还是copy代码，都太多了。

我一个个复制过来看吧。



```java
@AliasFor("basePackages")
String[] value() default {};

@AliasFor("value")
String[] basePackages() default {};
```

注释告诉我们，你可以这样写，`@ComponentScan("org.my.pkg")`，

你也可以这样写`@ComponentScan(basePackages = "org.my.pkg")`

因为value是个数组，所以你也可以这样写，`@ComponentScan(value = {"org.my.pkg","com.cn.test"})`

又因为加了`value` 和`basePackages`加了`@AliasFor`，所以你可以把当做是一样的效果。因此，`@ComponentScan(basePackages = {"org.my.pkg","com.cn.test"})`也是成立的。



```java
	Class<?>[] basePackageClasses() default {};
```

这个简单理解呢，就是上面的一种替代方案，他只能扫描这个类所在包下的所以被注解了的类。

比如，这么写：

```java
@ComponentScan(basePackageClasses = {AppConfig.class})
public class AppConfig {
}
```



```java
Class<? extends BeanNameGenerator> nameGenerator() default BeanNameGenerator.class;
```
这个呢也好理解，就是被注解的类，可能你没有指定beanName，那没关系，Spring可以帮你生成一个名字，生成规则是什么呢，大家也都知道。我就不在叙述了。



```java
	Class<? extends ScopeMetadataResolver> scopeResolver() default AnnotationScopeMetadataResolver.class;
```
这个我还真不知道，留着看源码时候解决吧。



```java
	ScopedProxyMode scopedProxy() default ScopedProxyMode.DEFAULT;
```
这个也不知道，留着。



```java

	
	String resourcePattern() default ClassPathScanningCandidateComponentProvider.DEFAULT_RESOURCE_PATTERN;
```

不知道，留着。



```java
	boolean useDefaultFilters() default true;

	Filter[] includeFilters() default {};

	Filter[] excludeFilters() default {};

	boolean lazyInit() default false;
```
useDefaultFilters，顾名思义，就是使用默认的过滤器。这个值挺重要的，在源码中可以清晰的看到他的作用，假如将他改为false，就扫描不到类`@Component`了。

lazyInit,控制懒加载的

includeFilters，excludeFilters 都是返回内部类Filter。因此看一下Filter

```java
@interface Filter {

    FilterType type() default FilterType.ANNOTATION;

    @AliasFor("classes")
    Class<?>[] value() default {};

    @AliasFor("value")
    Class<?>[] classes() default {};

    String[] pattern() default {};
}
```
简单理解就是，includeFilters，excludeFilters可以自定义过滤器，从而到达扫描的时候包含哪些类，不包含哪些类。哪怎么自定义过滤器呢，Spring提供了5中自定义过滤器的方式，其中有一种最容易理解。就是使用`TypeFilter`接口，配合`FilterTypoe.custom`的方式。
看如下代码
```java
@Retention(RetentionPolicy.RUNTIME)
public @interface Foo {
}

public class CustomFilter implements TypeFilter {
	@Override
	public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
		AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
		if (annotationMetadata.hasAnnotation(Foo.class.getName())) {
			return true;
		}
		return false;
	}
}

@Foo
public class DDD {
}

@ComponentScan(basePackageClasses = {AppConfig.class},
		includeFilters = @ComponentScan.Filter(classes = CustomFilter.class, type = FilterType.CUSTOM))
public class AppConfig {

}
```

这样，在运行的时候就可以通过@Foo注解将指定的类注入进来了。

pattern看官方注释就行。如果type是FilterType.AspectJ，那这个pattern就是AspectJ的解析表达式，如果type是FilterType.REGEX，那么pattern就是正则去匹配全类名。
