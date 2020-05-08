# initialize方法解析
#### 初始化SpringApplication（内部构造函数进行初始化工作，主要收集）

**initialize方法内部主要有以下几个功能**

1. 推断当前springboot所处的环境是否为WEB环境  

```java
this.webEnvironment = deduceWebEnvironment();
```

推断的依据为
```java
private static final String[] WEB_ENVIRONMENT_CLASSES = { "javax.servlet.Servlet",
			"org.springframework.web.context.ConfigurableWebApplicationContext" };

```
具体实现的方法
```java
private boolean deduceWebEnvironment() {
		for (String className : WEB_ENVIRONMENT_CLASSES) {
			if (!ClassUtils.isPresent(className, null)) {
				return false;
			}
		}
		return true;
	}
```

***判断当前环境是否为Web环境的依据是由WEB_ENVIRONMENT_CLASSES 中的两个类是否能从ClassLoader中获取，能返回true，否则false***  

2. 获得classLoader下所有继承自ApplicationContextInitializer.class类的实例  
getSpringFactoriesInstances方法实现了如何获得实例集合并排序
```java
private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type,
			Class<?>[] parameterTypes, Object... args) {
		ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
		// Use names and ensure unique to protect against duplicates
		// 首先获得names
		Set<String> names = new LinkedHashSet<String>(
				SpringFactoriesLoader.loadFactoryNames(type, classLoader));
		// 通过names获得对应的实例，根据以下方法
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
				classLoader, args, names);
		// 排序，两种方式可以介入排序功能1.使用@Order注解 2.实现Ordered接口
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
	}

```
根据name获取对应的实例，步骤为：
 - 获得实例class类
 - 获得构造方法，通常是无参的
 - 通过工具类构造函数生成实例  
 
 此为实际获取实例集合的相关方法
 ```java
private <T> List<T> createSpringFactoriesInstances(Class<T> type,
			Class<?>[] parameterTypes, ClassLoader classLoader, Object[] args,
			Set<String> names) {
		List<T> instances = new ArrayList<T>(names.size());
		for (String name : names) {
			try {
			    // 第一点
				Class<?> instanceClass = ClassUtils.forName(name, classLoader);
				Assert.isAssignable(type, instanceClass);
				// 第二点
				Constructor<?> constructor = instanceClass
						.getDeclaredConstructor(parameterTypes);
				// 第三点
				T instance = (T) BeanUtils.instantiateClass(constructor, args);
				instances.add(instance);
			}
			catch (Throwable ex) {
				throw new IllegalArgumentException(
						"Cannot instantiate " + type + " : " + name, ex);
			}
		}
		return instances;
	}
```
获得实例后，马上通过Order排序  

使用AnnotationAwareOrderComparator进行排序，从名字上判断是通过Order注解获取排序的参数  

![images](https://github.com/13129921509/free-files/blob/master/AnnotationAwareOrderComparator.jpg)   

从图中可得知，AnnotationAwareOrderComparator继承自OrderComparator  

> ***为什么有Ordered，还要有个Priority***  
> Priority并不是spring官方提供的注解，而是java官方提供的注解扩展库，与它类似的注解还有Resource注解,spring
完美适配了这些官方注解  

```java
/**
	 * This implementation checks for {@link Order @Order} or
	 * {@link javax.annotation.Priority @Priority} on various kinds of
	 * elements, in addition to the {@link org.springframework.core.Ordered}
	 * check in the superclass.
	 */
	protected Integer findOrder(Object obj) {
		// Check for regular Ordered interface 
		// 首先使用父类OrderComparator的findOrder方法检查对象是否实现Ordered接口，如果实现就返回具体order值
		Integer order = super.findOrder(obj);
		if (order != null) {
			return order;
		}

		// Check for @Order and @Priority on various kinds of elements
		// 此处IF主要判断对象的类型，因为Ordered注解可以被类，方法，属性所引用
		if (obj instanceof Class) {
		    // 判断obj上是否有	Order，Priority，如果实现就返回具体order值
			// 具体判断逻辑如下
			return OrderUtils.getOrder((Class<?>) obj);
		}
		else if (obj instanceof Method) {
			Order ann = AnnotationUtils.findAnnotation((Method) obj, Order.class);
			if (ann != null) {
				return ann.value();
			}
		}
		else if (obj instanceof AnnotatedElement) {
			Order ann = AnnotationUtils.getAnnotation((AnnotatedElement) obj, Order.class);
			if (ann != null) {
				return ann.value();
			}
		}
		else if (obj != null) {
			order = OrderUtils.getOrder(obj.getClass());
			if (order == null && obj instanceof DecoratingProxy) {
				order = OrderUtils.getOrder(((DecoratingProxy) obj).getDecoratedClass());
			}
		}

		return order;
	}

```
getOrder方法主要分俩步走
```java
/**
	 * Return the order on the specified {@code type}, or the specified
	 * default value if none can be found.
	 * <p>Takes care of {@link Order @Order} and {@code @javax.annotation.Priority}.
	 * @param type the type to handle
	 * @return the priority value, or the specified default order if none can be found
	 * @see #getPriority(Class)
	 */
	public static Integer getOrder(Class<?> type, Integer defaultOrder) {
	    // 能通过在对象上获得Order注解上的值，就直接返回获得值
		Order order = AnnotationUtils.findAnnotation(type, Order.class);
		if (order != null) {
			return order.value();
		}
		// 否则，接着是否带有Priority注解，如果有直接返回值
		Integer priorityOrder = getPriority(type);
		if (priorityOrder != null) {
			return priorityOrder;
		}
		// 都没有就返回默认值
		return defaultOrder;
	}

```

从以上方法看，假如对象上同时带有Ordered与Priority注解，从代码执行顺序来看只会返回Ordered的值，毕竟是亲儿子

AnnotationAwareOrderComparator类主要作用是手机Order或Priority的value，最终sort的还是根据OrderCompartor中compare方法实现  
```java
private int doCompare(Object o1, Object o2, OrderSourceProvider sourceProvider) {
		boolean p1 = (o1 instanceof PriorityOrdered);
		boolean p2 = (o2 instanceof PriorityOrdered);
		if (p1 && !p2) {
			return -1;
		}
		else if (p2 && !p1) {
			return 1;
		}

		// Direct evaluation instead of Integer.compareTo to avoid unnecessary object creation.
		int i1 = getOrder(o1, sourceProvider);
		int i2 = getOrder(o2, sourceProvider);
		return (i1 < i2) ? -1 : (i1 > i2) ? 1 : 0;
	}

```

3. 获得classLoader下所有继承自ApplicationContextInitializer.class类的实例  

获得的方式跟获得ApplicationContextInitializer.class一样是通过getSpringFactoriesInstances方法获取 

#### 总结  

initialize方法主要是初始化SpringApplication中的相关参数，为run方法做准备，
在以上代码中，出现了很多spring提供的官方攻击类，比如：
- PropertiesLoaderUtils
- AnnotationUtils
- ReflectionUtils 
- ClassUtils  
这些工具包提供了相当多方便的方法，有空可以学习下

**下一篇主要讲解SpringApplication.run方法,难度比较大，需要时间沉淀**
