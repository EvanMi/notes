# spring

## springboot

### servlet原生

#### servlet注册

1、@WebServlet + @ServletComponentScan

2、@Bean 等方式注册为spring的原生

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

## idea spring嵌入容器的无法定位modal中的webapp目录bug

springboot中是通过DocumentRoot中的getCommonDocumentRoot来获取嵌入式容器的baseDir的。具体体现在

TomcatServletWebServerFacotry中如果DocumentRoot为空，会定向docbase到一个临时目录。所以在idea中可以通过

WebServerFactoryCustomizer  # addContextCustomizers(context -> {context.setDocBase("...")}) 来将路径指定到modal中。

