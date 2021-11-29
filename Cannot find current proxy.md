# IllegalStateException: Cannot find current proxy: Set 'exposeProxy' property on Advised to 'true'
在使用 @Async 注解实现异步线程的时候，为了能够在同类中调用，使用AopContext获取类的实例，结果报错：

调用如下：

```java
@GetMapping("test03") 
public void testAsync03() throws InterruptedException
{
    log.info("=====03主线程执行start: " + Thread.currentThread().getName());
    AsyncTestController currentProxy = (AsyncTestController) AopContext.currentProxy();
    currentProxy.doAsync01();
    currentProxy.doAsync02();
    log.info("=====03主线程执行end: " + Thread.currentThread().getName());
}

@Async("defaultTaskExecutor")
public void doAsync01() throws InterruptedException
{
    Thread.sleep(3000);
    log.info("=====子线程001执行: " + Thread.currentThread().getName());
}

@Async("defaultTaskExecutor")
public void doAsync02()
{
    log.info("=====子线程002执行: " + Thread.currentThread().getName());
}
```

结果如下：

![](https://img-blog.csdnimg.cn/20191209224344104.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDkxMDM3Mg==,size_16,color_FFFFFF,t_70)

但是回头看看我的配置完全是OK的：

    @EnableAspectJAutoProxy(proxyTargetClass = true, exposeProxy = true)


### 问题分析

**找错的常用方法：逆推法。** 首先我们找到报错的最直接原因：`AopContext.currentProxy()`这句代码报错的，因此有必要看看`AopContext`这个**工具类**：

```java
public final class AopContext
{
	private static final ThreadLocal < Object > currentProxy = new NamedThreadLocal("Current AOP proxy");
	private AopContext()
	        {
	}
	// 该方法是public static方法，说明可以被任意类进行调用    
	public static Object currentProxy() throws IllegalStateException
	        {
		Object proxy = currentProxy.get();
		// 它抛出异常的原因是当前线程并没有绑定对象        
		// 而给线程版定对象的方法在下面：特别有意思的是它的访问权限是default级别，也就是说只能Spring内部去调用~        
		if(proxy == null)
		            {
			throw new IllegalStateException("Cannot find current proxy: Set 'exposeProxy' property on Advised to 'true' to make it available.");
		} else
		            {
			return proxy;
		}
	}
	// 它最有意思的地方是它的访问权限是default的，表示只能给Spring内部去调用~    
	// 调用它的类有CglibAopProxy和JdkDynamicAopProxy    
	@Nullable 
	  static Object setCurrentProxy(@Nullable Object proxy)
	    {
		Object old = currentProxy.get();
		if(proxy != null)
		        {
			currentProxy.set(proxy);
		} else
		        {
			currentProxy.remove();
		}
		return old;
	}
}
```

 从此工具源码可知，决定是否抛出所示异常的直接原因就是请求的时候`setCurrentProxy()`方法是否被调用过。通过寻找发现只有两个类会调用此方法，并且都是Spring内建的类且都是代理类的**处理类**：`CglibAopProxy`和`JdkDynamicAopProxy.`

> 说明：本文所有示例，都基于接口的代理，所以此处只以`JdkDynamicAopProxy`作为代表进行说明即可.

我们知道在执行代理对象的目标方法的时候，都会交给`InvocationHandler`处理，因此做事情的在`invoke()`方法里：

```java
@Nullable    
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  Object oldProxy = null;        
  boolean setProxyContext = false;        
  TargetSource targetSource = this.advised.targetSource;        
  Object target = null;
  ...         
  if (this.advised.exposeProxy) {            
    oldProxy = AopContext.setCurrentProxy(proxy);            
    setProxyContext = true;
  }
  ...
}
```

 so，最终决定是否会调用set方法是由`this.advised.exposeProxy`这个值决定的，因此下面我们只需要关心`ProxyConfig.exposeProxy`这个属性值什么时候被赋值为true的就可以了。

> `ProxyConfig.exposeProxy`这个属性的默认值是false。其实最终调用设置值的是同名方法`Advised.setExposeProxy()`方法，而且是通过反射调用的.

`@EnableAspectJAutoProxy(exposeProxy = true)`的作用

此注解它导入了`AspectJAutoProxyRegistrar`，最终`设置`此注解的两个属性的方法为：

```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {
	AspectJAutoProxyRegistrar() {
	}
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
		AnnotationAttributes enableAspectJAutoProxy = AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
		if (enableAspectJAutoProxy != null) {
			if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
				AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
			}
			if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
				AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
			}
		}
	}
}
public abstract class AopConfigUtils {
	...    public static void forceAutoProxyCreatorToUseClassProxying(BeanDefinitionRegistry registry) {
		if (registry.containsBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator")) {
			BeanDefinition definition = registry.getBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator");
			definition.getPropertyValues().add("proxyTargetClass", Boolean.TRUE);
		}
	}
	public static void forceAutoProxyCreatorToExposeProxy(BeanDefinitionRegistry registry) {
		if (registry.containsBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator")) {
			BeanDefinition definition = registry.getBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator");
			definition.getPropertyValues().add("exposeProxy", Boolean.TRUE);
		}
	}
	...
}
```

看到此注解标注的属性值最终都被设置到了`internalAutoProxyCreator`身上，也就是进而重要的一道菜：自动代理创建器。

在此各位小伙伴需要先明晰的是：`@Async`的代理对象并不是由自动代理创建器来创建的，而是由`AsyncAnnotationBeanPostProcessor`一个单纯的`BeanPostProcessor`实现的。

这让我想起了类似的注解： @EnableTransactionManagement ，该注解向容器注入的是自动代理创建器`InfrastructureAdvisorAutoProxyCreator`，所以`exposeProxy = true`对它的代理对象都是生效的，因此可以正运行，另外，@EnableCaching也是可以的。

`@EnableAsync`给容器注入的是`AsyncAnnotationBeanPostProcessor`，它用于给`@Async`生成代理，但是它仅仅是个`BeanPostProcessor`并不属于自动代理创建器，因此`exposeProxy = true`对它无效。 所以`AopContext.setCurrentProxy(proxy);`这个set方法肯定就不会执行，so但凡只要业务方法中调用`AopContext.currentProxy()`方法就铁定抛异常。

将@Transaction 和 @Async 放在一起使用：

```java
@GetMapping("test03")    
public void testAsync03() throws InterruptedException {
	log.info("=====03主线程执行start: " + Thread.currentThread().getName());
	AsyncTestController currentProxy = (AsyncTestController) AopContext.currentProxy();
	currentProxy.doAsync01();
	currentProxy.doAsync02();
	log.info("=====03主线程执行end: " + Thread.currentThread().getName());
}
@Async("defaultTaskExecutor")    
@Transactional    
public void doAsync01() throws InterruptedException {
	Thread.sleep(3000);
	log.info("=====子线程001执行: " + Thread.currentThread().getName());
}
@Async("defaultTaskExecutor")    
@Transactional    
public void doAsync02() {
	log.info("=====子线程002执行: " + Thread.currentThread().getName());
}
}
```

运行结果如下：

![](https://img-blog.csdnimg.cn/20191216174542187.png)

这个示例的结论，**相信是很多小伙伴都没有想到的**。仅仅只是加入了事务，`@Asycn`竟然就能够完美的使用`AopContext.currentProxy()`获取当前代理对象了。

为了便于理解，我分步骤讲述如下，不出意外你肯定就懂了：

1. `AsyncAnnotationBeanPostProcessor`在创建代理时有这样一个逻辑：若已经是`Advised`对象了，那就只需要把`@Async`的增强器添加进去即可。若不是代理对象才会自己去创建

   public abstract class AbstractAdvisingBeanPostProcessor extends ProxyProcessorSupport implements BeanPostProcessor {
   	@Override	
   	public Object postProcessAfterInitialization(Object bean, String beanName) {
   		if (bean instanceof Advised) {
   			advised.addAdvisor(this.advisor);
   			return bean;
   		}
   		// 上面没有return，这里会继续判断自己去创建代理~
   	}
   }

2. 自动代理创建器`AbstractAutoProxyCreator`它实际也是个`BeanPostProcessor`，所以它和上面处理器的执行顺序很重要~~~

3. 两者都继承自`ProxyProcessorSupport`所以都能创建代理，且实现了`Ordered`接口 1. `AsyncAnnotationBeanPostProcessor`默认的order值为`Ordered.LOWEST_PRECEDENCE`。但可以通过`@EnableAsync`指定order属性来改变此值。 执行代码语句：`bpp.setOrder(this.enableAsync.<Integer>getNumber("order"));` 2. `AbstractAutoProxyCreator`默认值也同上。但是在把自动代理创建器添加进容器的时候有这么一句代码：`beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);` 自动代理创建器这个处理器是`最高优先级`

4. 由上可知因为标注有`@Transactional`，所以自动代理会生效，因此它会先交给`AbstractAutoProxyCreator`把代理对象生成好了，再交给后面的处理器执行

5. 由于`AbstractAutoProxyCreator`先执行，所以`AsyncAnnotationBeanPostProcessor`执行的时候此时Bean已经是代理对象了，由步骤1可知，此时它会沿用这个代理，只需要把切面添加进去即可~

从上面步骤可知，加上了事务注解，最终代理对象是由自动代理创建器创建的，因此`exposeProxy = true`对它有效，这是解释它能正常work的最为根本的原因。`@Transactional`只为了创建代理对象而已，所在放在哪儿对`@Async`的作用都不会有本质的区别。

解决方案
----

对上面现象原因可以做一句话的总结：**`@Async`要想顺利使用`AopContext.currentProxy()`获取当前代理对象来调用本类方法，需要确保你本Bean已经被自动代理创建器`提前代理`。**

> 在实际业务开发中：只要的类标注有`@Transactional`或者`@Caching`等注解，就可以放心大胆的使用吧

知晓了原因，解决方案从来都是信手拈来的事。 不过如果按照如上所说需要`隐式依赖`这种方案我**非常的不看好**，总感觉不踏实，也总感觉报错迟早要来。（**比如某个同学该方法不要事务了/不要缓存了，把对应注解摘掉就瞬间报错了**，到时候你可能哭都没地哭诉去~）

> 备注：`墨菲定律`在开发过程中从来都没有不好使过~~~程序员兄弟姐妹们应该深有感触吧

下面根据我个人经验，介绍一种解决方案中的最佳实践：

> 遵循的最基本的原则是：显示的指定比隐式的依赖来得更加的靠谱、稳定

### 终极解决方案：

```java
@Component
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
	@Override    
  public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		BeanDefinition beanDefinition = beanFactory.getBeanDefinition(TaskManagementConfigUtils.ASYNC_ANNOTATION_PROCESSOR_BEAN_NAME);
		beanDefinition.getPropertyValues().add("exposeProxy", true);
	}
}
```

**这样我们可以在`@Async`和`AopContext.currentProxy()`就自如使用了，不再对别的啥的有依赖性~**

> 其实我认为最佳的解决方案是如下两个（都需要Spring框架做出修改）：
>
>  1、`@Async`的代理也交给自动代理创建器来完成
>
>  2、`@EnableAsync`增加`exposeProxy`属性，默认值给false即可（**此种方案的原理同我示例的最佳实践~**）

### 总结

最后再总结两点，小伙伴们使用的时候稍微注意下就行：

1.  请不要在异步线程里使用`AopContext.currentProxy()`
2.  `AopContext.currentProxy()`不能使用在非代理对象所在方法体内