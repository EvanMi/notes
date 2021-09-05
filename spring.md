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

### 运行阶段

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

