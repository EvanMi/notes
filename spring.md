# spring

## springboot

### servlet原生

#### servlet注册

-1、通过web.xml来进行注册

0、通过ServletContext.add来进行注册，在ServletContextInitializer中会用到。

1、@WebServlet + @ServletComponentScan

~~2、@Bean 等方式注册为spring的原生~~

3、使用对应的RegistrationBean 例如 ServletRegistrationBean

#### 异步非阻塞的servlet

异步：javax.servlet.ServletRequest#startAsync()

​			javax.servlet.AsyncContext()

非阻塞：javax.servlet.ServletInputStream#setReadListener

​						javax.servlet.ReadListener

​				javax.servlet.ServletOutputStream#setWriteListener

​						javax.servlet.WriteListener

```java
@WebServlet(urlPatterns="/test", asyncSupported=true)
public class MyServlet extends HttpServlet {
  protected void doGet(HttpServletRequest req, HttpServletResponse resp) {
    AsyncContext asyncContext = req.startAsync();
    asyncContext.start(() -> {
      ////
      asyncContext.complete();//这是必须要的
    });
  }
}
```



### 配置文件

#### 使用配置

1、XML中使用$占位符 结合 PropertyPlaceholderConfigurer (spring时代) springboot时代就不需要PropertyPlaceholderConfigurer了，直接使用$占位符 + @ImportResource注解即可。

2、@Value可以作用在field上，也可以使用在方法的参数上。

3、通过Environment来获取,可以结合Binder来进行使用。

4、@ConfigurationProperties 可以支持静态内部类中的属性嵌套赋值，可以支持spring自带的validation，配合 @Validated来实现。

#### 自定义的时机

在Spring Framework中应该在AbstractApplicationContext#prepareBeanfactory方法前。

在SpringBoot中，应该在SpringApplication#refreshContext方法前。

在Environment中会封装很多PropertySource

#### 自定义外部化配置

1、基于SpringApplicationRunListener#environmentPrepared

2、基于ApplicationEnvironmentPreparedEvent扩展(有ApplicationRunListener的默认实现EventPublishingRunListener在environmentPrepared发出，通过自己在spring.factories中定义自己的事件监听器来获取相应的事件)

3、基于EnvironmentPostProcessor扩展

4、基于ApplicationContextIntializer扩展

5、基于SpringApplicationRunListener#contextPropered扩展

6、基于ApplicationPreparedEvent扩展(实现方式同2)

#### 配置文件查找位置

1、classpath 根路径

2、classpath 根路径下config目录

3、jar包当前目录

4、jar包当前目录的config目录

5、/config目录和第一层子目录

#### 配置文件加载顺序

1、当前jar包内部的application.properties 和 application.yml

2、当前jar包内部的application-{profile}.properties 和 application-{profile}.yml

3、引用外部jar包的application.properties 和 application.yml

4、引用的外部jar包的application-{profile}.properties 和 application-{profile}.yml

后面的可以覆盖前面的同名配置项。

### springboot启动流程

#### SpringApplication的运行方式

（1）直接使用SpringApplication.run(..)

（2）new SpringApplication() 然后设置各种参数，最后执行run

（3)  new SpringApplicationBuilder() 可以使用fluent API来设置各种参数，最后执行run

#### 准备阶段

（1）配置 springboot bean来源

（2）Web应用类型推断

servelet的方式优先于reactive的方式。

（3）推断主类

（4）加载应用上下文初始化器（ApplicationContextInitializer）

应用上下文初始化器保存在META-INF/spring.factories中，通过SpringFacotoriesLoader来进行加载。

（5）加载应用事件监听器（ApplicationListener）

应用事件监听器保存在META-INF/spirng.factories中，通过SpringFacotoriesLoader来进行加载。

#### 运行阶段

（1）加载“SpringApplication运行”监听器（SpringApplicationRunListener）

应用事件监听器保存在META-INF/spirng.factories中，通过SpringFacotoriesLoader来进行加载。

构造方法中必须有 SpringApplication 和 String[] 类型的两个参数，springboot会默认传进来，不传进来会报错。

默认的一个实现为 EventPublishingRunListener

就是这个Listener内部会持有一个SimpleApplicationEventMulticaster，然后将上一步中（5）中的SrpingListener都加进来给他们发送事件。

（2）运行"SrpingApplication运行"监听器

（3）监听SpringBoot事件、Spring事件

（4）创建应用上下文、Environment

Web Reactive：AnnotationConfigReactiveWebServerApplicationContext  StandardEnvironment

Web Servlet: AnnotationConfigServletWebServerApplicationContext StandardServletEnvironment

非web: AnnotationConfigApplicationContext StandardEnvironment

（5）失败后的故障分析报告

（6）回调 CommandLineRunner 、ApplicationRunner

### @ConfigurationProperties的使用方法

（1）@EnableConfigurationProperties + @ConfigurationProperties

```
@ConfigurationProperties(prefix = "service.properties")
public class HelloServiceProperties {
    private static final String SERVICE_NAME = "test-service";

    private String msg = SERVICE_NAME;
       set/get
}


@Configuration
@EnableConfigurationProperties(HelloServiceProperties.class)
@ConditionalOnClass(HelloService.class)
@ConditionalOnProperty(prefix = "hello", value = "enable", matchIfMissing = true)
public class HelloServiceAutoConfiguration {

}

@RestController
public class ConfigurationPropertiesController {

    @Autowired
    private HelloServiceProperties helloServiceProperties;

    @RequestMapping("/getObjectProperties")
    public Object getObjectProperties () {
        System.out.println(helloServiceProperties.getMsg());
        return myConfigTest.getProperties();
    }
}

```

（2）@Component + @ConfigurationProperties

## ServletContainerInitializer

在Servlet3.0标准的基础上Spring包装了自己的Initializer

```
@HandlesTypes(WebApplicationInitializer.class) ## 所有这个类型的class都会被识别
SpringServletContainerInitializer implements ServletContainerInitializer

在上面的基础上，spring有提供了一些封装
编程驱动 AbstractDispatcherServletInitializer
注解驱动 AbstractAnnotationConfigDispatcherServletIntializer
springboot提供的扩展
SpringBootServletInitializer
```

## springmvc

### View 请求流程

![alt](imgs/spring_mvc_process.png)

### View视图协商

![alt](imgs/springmvc_view_negotiation.png)

客户端指定媒体类型例子：

（1）Accept请求头  Accept:text/html

（2）请求查询参数 /path?format=pdf

（3）路径扩展名 /abc.pdf

最佳匹配规则：

（1）manager解析出所有的request中可用的mediaType

（2）使用有所的ViewResolveer进行以及mediaType进行视图解析，获取到所有可用的View

（3）最后根据匹配程度选取第一个符合的View

### Rest请求流程

![alt](imgs/springmvc_rest_process.png)

### Rest内容协商流程

![alt](imgs/spring_mvc_rest_negotiation.png)



### Rest中使用到的各种量

#### 请求

| 注解           | 说明                                 |
| -------------- | ------------------------------------ |
| @RequestParam  | 获取请求参数                         |
| @RequestHeader | 获取请求头                           |
| @CookieValue   | 获取Cookie值                         |
| @RequestBody   | 获取完整请求主体内容                 |
| @PathVariable  | 获取请求路径变量                     |
| RequestEntity  | 获取请求内容（包括请求主体和请求头） |

#### 相应

| 注解           | 说明                             |
| -------------- | -------------------------------- |
| @ResponseBody  | 相应主体注解声明                 |
| ResponseEntity | 相应内容（包括相应主体和响应头） |
| ResponseCookie | 相应Cookie内容                   |

#### 拦截

| 注解                  | 说明                        |
| --------------------- | --------------------------- |
| @RestControllerAdvice | @RestController注解切面通知 |
| HandlerInterceptor    | 处理方法拦截器              |

#### 跨域

| 注解                             | 说明             |
| -------------------------------- | ---------------- |
| @CrossOrigin                     | 资源跨域声明注解 |
| CorsFilter                       | 资源跨域拦截器   |
| WebMvcConfigurer#addCorsMappings | 注册资源跨域信息 |



## idea spring嵌入容器的无法定位modal中的webapp目录bug

springboot中是通过DocumentRoot中的getCommonDocumentRoot来获取嵌入式容器的baseDir的。具体体现在

TomcatServletWebServerFacotry中如果DocumentRoot为空，会定向docbase到一个临时目录。所以在idea中可以通过

WebServerFactoryCustomizer  # addContextCustomizers(context -> {context.setDocBase("...")}) 来将路径指定到modal中。

## Servlet 核心API

| 核心组件API                               | 说明                         | 起始版本 | SpringFramework代表实现           |
| ----------------------------------------- | ---------------------------- | -------- | --------------------------------- |
| javax.servlet.Servlet                     | 动态内容组件                 | 1.0      | DispatcherServlet                 |
| javax.servlet.Filter                      | Servlet过滤器                | 2.3      | CharacterEncodingFilter           |
| javax.servlet.ServletContext              | Servlet应用上下文            |          |                                   |
| javax.servlet.AsyncContext                | 异步上下文                   | 3.0      | 无                                |
| javax.servlet.ServletContextListener      | ServletContext生命周期监听器 | 2.3      | ContextLoaderListener             |
| javax.servlet.ServletRequestListener      | ServletRequest生命周期监听器 | 2.3      | RequestContexyListener            |
| javax.servlet.http.HttpSessionListener    | HttpSession生命周期监听器    | 2.3      | HttpSessionMutexListener          |
| javax.servlet.AsyncListener               | 异步上下文监听器             | 3.0      | StandardServletAsyncWebRequest    |
| javax.servlet.ServletContainerInitializer | Servlet容器初始化器          | 3.0      | SptingServletContainerInitializer |

## dispatcherServlet层次

HttpServlet

​	HttpServletBean

​			# init()方法中会读取配置并设置相关属性。

​		FrameServlet

​			在这里会初始化一个spring容器，并将父容器设置为spring容器。

​			DispatcherServlet

## springboot 嵌入式 Servlet容器限制

### 容器限制

（1）不支持 web.xml部署

​		可以通过RegistrationBean 或 @Bean注解

（2）不支持ServletContainerInitializer接口

​		使用 ServletContextInitializer

（3）注解驱动限制

​		需要结合 @ServletComponentScan

### springboot Servlet注册

​	ServletContextInitializer

​		RegistrationBean

​			ServletListenerRegistrationBean

​				@WebListener

​			FilterRegistrationBean

​				@WebFilter

​			ServletRegistrationBean

​				@WebServlet

@ServletComponentScan扫描package -> @Web* -> RegistrationBean Bean定义 -> RegistartionBean Bean

## 理解Reactive

（1）维基百科

```
Reactive programming is a declarative programing paradigm concerned with data streams and the propagation of change.With this paradigm it is possible to express static(e.g. arrays) or dynamic(e.g. event emitters) data streams with ease, and also communicate that an inferred dependency within the associated execution model exists, which facilitates the automatic propagation of the changed data flow.
```

（2）The Reactive Manifesto

```
Reactive Systems are: Responsive, Resilient, Elastic and Message Driven.
```

（3）Spring Framework

```
The term "reactive" refers to programming models that are built around reacting to change--network component reacting to I/0 events, UI controller reacting to mouse events, etc. In that sense non-blocking is reactive because instead of being blocked we are now in the mode of reacting to notifications as operations complete or data becomes available.
```

（4）ReactiveX

```
ReactiveX extends the observer pattern to support sequences of data and/or events and adds operators that allow you to compose sequences together declaratively while abstracting away concerns about things like lowlevel threading, synchronization, thread-safety, concurrent data structures, and non-blocking I/O.
```

（5）Reactor

```
The reactor programming paradigm is often presented in object-oriented languages as an extension of the Observer design pattern. One can also compare the main reactive streams pattern with the familiar iterator design pattern, as there is a duality to the iterable-iterator pair in all of these libraries. One major difference is that,while an iterator is pull-based, reactive streams are push-based.
```

（6）@andrestaltz（著名作者)

```
In a way, this isn't anything new. Event buses or your typical click events are really an asynchronous event stream, on whitch you can observe and do some side effects. Reactive is that idea on steroids, You are able to create data streams of anything, not just from click and hover events. Streams are cheap and unbiquitous, anything can be a stream:variables, user inputs, properties, caches, data structures, etc.
```

### Recative Streams规范

Reactive Streams is a standard and specification for Stream-oriented libraries for the JVM that

- process a potentially unbounded number of elements
- in sequence
- asynchronously passing elements between components
- with mandatory non-blocking backpressure

### webflux请求流程

![alt](imgs/webflux_request_process.png)

## spring aop 个中执行顺序

```
@Order(1)
Aspect1 {...}

@Oder(2)
Aspect2 {...}
```

在以上请款下的执行顺序：

Aspect1 : Around

Aspect1 : Before

Aspect2 : Around

Aspect2 : Before

被切入的方法本体执行

Aspect2: Around

Aspect2: After

Aspect2: AfterReturning

Aspect1 : Around

Aspect1 : After

Aspect1 : AfterReturning

## Spring IOC

IOC的作用其实类似于一种声明式的编程，我们声明了需要某个对象，然后由“数据源”来提供具体的对象，我们只管使用该对象来实现相应的功能即可。而在spring中spring框架就是一个“数据源”，通过依赖注入的方式将我们的声明替换为具体的对象。

### 依赖注入 和 依赖查找

IOC的实现方式很多，其中最常见的两种就是依赖注入和依赖查找。

| 类型     | 依赖处理 | 实现便利性 | 代码侵入性   | API依赖性     | 可读性 |
| -------- | -------- | ---------- | ------------ | ------------- | ------ |
| 依赖查找 | 主动获取 | 相对繁琐   | 侵入业务逻辑 | 依赖容器API   | 良好   |
| 依赖注入 | 被动提供 | 相对便利   | 低侵入性     | 不依赖容器API | 一般   |

(1)依赖查找

按名称查找

- 实时查找（通过id 来getBean)
- 延迟查找（也就是把对象包装在objectFactory中，每次get都由objectFactory来延迟查找）

按类型查找

- 单个Bean对象
- 集合Bean对象

根据名称 + 类型进行查找

根据java注解查找（在beanFactory中有对象的方法了）

- 单个Bean对象
- 集合Bean对象

（2）依赖注入

根据Bean名称注入

根据Bean类型注入

- 单个Bean对象
- 集合Bean对象

注入容器内建Bean对象

注入非Bean对象

注入类型

- 实时注入
- 延迟注入

（3）依赖来源

- 自定义Bean
- 容器内建Bean（Spring注册的Bean）
- 容器内建依赖（比如BeanFactory，通过getBean是无法获取到的）

### 注册 Spring Bean

- XMl配置元信息

<bean name="..." ... />

- Java注解配置元信息

@Bean @Component @Import

- Java API配置元信息

命名方式 BeanDefinitionRegistry#registerBeanDefinition(String,BeanDefiniton)

非命名方式 BeanDefinitonReaderUtils#registerWithGeneratedName(AbstractBeanDefinition, BeanDefinitionRegistry)

配置方式 AnnotatedBeanDefinitionReader#gegister(Class...) 就是指定一个配置类

### 传统IoC容器的实现

1、Java Beans作为IoC容器

特性：依赖查找、生命周期管理、配置元信息、事件、自定义、资源管理、持久化

内省(Introspector)是专门用来操作JavaBean属性的。不是所有的字段(Field)都能被称之为属性，只有某些字段具有getXXX或setXXX方法时才能称之为属性，当然要称为是一个Bean还需要有一个无惨的构造器，而内省就是对这些属性进行操作。

```java
Person p = new Person();
BeanInfo beanInfo = Introspector.getBeanInfo(p.getClass());
PropertyDescriptor[] pds = beanInfo.getPropertyDescriptors();
        for(PropertyDescriptor pd:pds) {
            System.out.println(pd.getName());
            System.out.println(pd.getPropertyType());
        }

PropertyDescriptor pd = new PropertyDescriptor("age", Person.class);
Person p = new Person();
Method setAgeMethod = pd.getWriteMethod();
setAgeMethod.invoke(p,25);
Method getAgeMethod = pd.getReadMethod();
System.out.println(getAgeMethod.invoke(p, null));
```

### ApplicationContext

ApplicationContext除了IoC容器角色，还有提供：

- 面向切面(AOP)
- 配置元信息(Configuration Metadata)
- 资源管理(Resources)
- 事件(Events)
- 国际化(I18n)
- 注解(Annoations)
- Environment抽象(Environment Abstraction)



