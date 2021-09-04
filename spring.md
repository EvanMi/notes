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

（1）配置 spring bean来源



