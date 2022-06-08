## springboot的方式集成camel 和 hawtio

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <dependency>
            <groupId>io.hawt</groupId>
            <artifactId>hawtio-springboot</artifactId>
            <version>2.9.1</version>
            <exclusions>
                <exclusion>
                    <artifactId>log4j-to-slf4j</artifactId>
                    <groupId>org.apache.logging.log4j</groupId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.apache.camel.springboot</groupId>
            <artifactId>camel-management-starter</artifactId>
            <version>3.16.0</version>
        </dependency>
        <!--相关依赖-->
```

```yaml
#配置文件
management:
  endpoints:
    web:
      exposure:
        include:
          - hawtio
          - jolokia

hawtio:
  authenticationEnabled: false
```

```java
package com.jd.jnos.baize.config;

import org.springframework.boot.actuate.autoconfigure.jolokia.JolokiaEndpoint;
import org.springframework.boot.actuate.autoconfigure.security.servlet.EndpointRequest;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.builders.WebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.web.csrf.CookieCsrfTokenRepository;

@Configuration
public class BaizeSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
                .passwordEncoder(new BCryptPasswordEncoder())
                .withUser("baize") // 添加用户admin
                .password(new BCryptPasswordEncoder().encode("Baize123#@!"))
                .roles("ADMIN");// 添加角色为admin，user
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/actuator/**").hasRole("ADMIN")
                .anyRequest().authenticated()
                .and()
                .formLogin().and()
                .httpBasic().and()
                .csrf().csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .ignoringRequestMatchers(EndpointRequest.to(JolokiaEndpoint.class));
    }

    @Override
    public void configure(WebSecurity webSecurity)  {
        webSecurity.ignoring().antMatchers("/api/**", "/system/**");
    }
}

```

spring security的WebSecurity和HttpSecurity区别

https://stackoverflow.com/questions/62677271/httpsecurity-permitall-and-websecurity-ignoring-functions-for-un-auth-urls

## Camel 启动和删除路由

```java
ExtendedCamelContext adapt = camelContext.adapt(ExtendedCamelContext.class);
final RoutesLoader loader = adapt.getRoutesLoader();
ExecutorService executorService = Executors.newSingleThreadExecutor();
executorService.submit(() -> {
    int count = 0;
    while (true) {
        if (count % 2 == 0) {
            log.info("添加了xml");
            final Resource resource = ResourceHelper.fromString("camel.xml", xml);
            loader.loadRoutes(resource);
        } else {
            log.info("删除了xml");
            SpringBootCamelContext springBootCamelContext = camelContext.adapt(SpringBootCamelContext.class);
            springBootCamelContext.stopRoute("test001");
            boolean test001 = springBootCamelContext.removeRoute("test001");
            log.info("{}", test001);

        }
        count++;
        TimeUnit.SECONDS.sleep(30);
    }
});
```



```java
static class MyClassLoader extends ClassLoader {
    public MyClassLoader(ClassLoader parent) {
        super(parent);
    }
}

@Data
static class A {
    private Object a;
}

public static void main(String[] args) throws Exception {
    ClassPool pool = ClassPool.getDefault();
    //构建interface
        CtClass c2 = pool.makeInterface("com.jd.Abc");
        CtMethod ctMethod = new CtMethod(pool.getCtClass("java.util.Map"), "test",
                new CtClass[] {pool.getCtClass("java.util.Map")}, c2);
        ctMethod.setModifiers(Modifier.PUBLIC);
        ctMethod.setModifiers(Modifier.ABSTRACT);
        c2.addMethod(ctMethod);
    MyClassLoader myClassLoader = new MyClassLoader(JsfComponent.class.getClassLoader());
    System.out.println(myClassLoader.getParent());
    Class aClass = c2.toClass(myClassLoader, null);
    Object o = Proxy.newProxyInstance(myClassLoader, new Class[]{aClass}, new InvocationHandler() {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            return "aaaaaaaaa";
        }
    });
    A a = new A();
    a.setA(o);
    System.out.println(a.getA().toString());
    System.out.println(a.getClass().getClassLoader());
    System.out.println(a.getA().getClass().getClassLoader());

    MyClassLoader myClassLoader2 = new MyClassLoader(JsfComponent.class.getClassLoader());
    aClass = c2.toClass(myClassLoader2, null);
    o = Proxy.newProxyInstance(myClassLoader2, new Class[]{aClass}, new InvocationHandler() {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            return "aaaaaaaaa";
        }
    });
    a.setA(o);
    System.out.println(a.getA().toString());
    System.out.println(a.getClass().getClassLoader());
    System.out.println(a.getA().getClass().getClassLoader());

    String path = "/Users/mipengcheng3/works/maven/repo/com/jd/jnos/baize/baize-hub-rpc-api/1.0.0-SNAPSHOT/baize-hub-rpc-api-1.0.0-SNAPSHOT.jar";
    URL url = new File(path).toURI().toURL();
    URLClassLoader urlClassLoader = new URLClassLoader(new URL[]{url}, JsfComponent.class.getClassLoader());
    Class<?> aClass1 = urlClassLoader.loadClass("com.jd.jnos.baize.api.service.BaizeApplicationRelServiceJsf");
    Class<?> aClass2 = urlClassLoader.loadClass("com.jd.jnos.baize.api.model.request.BaizeApplicationRelRequest");
    Object o2 = Proxy.newProxyInstance(urlClassLoader, new Class[]{aClass1}, new InvocationHandler() {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

            return null;
        }
    });
    Object xxx = aClass2.newInstance();
    new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println(a.getA() + "===========");
            System.out.println(o2.toString());
            for (Method method : aClass1.getMethods()) {
                try {
                    method.invoke(o2,xxx);
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                } catch (InvocationTargetException e) {
                    e.printStackTrace();
                }
            }
        }
    }).start();

    TimeUnit.SECONDS.sleep(10);
}
```

## Camel 拦截器

// ================  因为RoutePolicy是基于Route的, 所以为了做到全局统一配置, 这里我们通过实现RoutePolicyFactory接口来实现
// RoutePolicyFactoryInstrumentationImpl将为用户自定义的每个Route添加一个RoutePolicyAdvice, 实现了类似切面的逻辑统一存放。

```java
class RoutePolicyFactoryInstrumentationImpl implements RoutePolicyFactory {
	static final String EXCHANGE_PROERTIES_KEY_STOPWATCH_FOR_METRIC = "METRIC-STOPWATCH";
private final RoutePolicy policy = new RoutePolicyInstrumentationImpl();

@Override
public RoutePolicy createRoutePolicy(CamelContext camelContext, String routeId, RouteDefinition route) {
	return policy;
}

// =========================================
static class RoutePolicyInstrumentationImpl extends org.apache.camel.support.RoutePolicySupport
      implements RoutePolicy {

	@Override
	public void onExchangeBegin(Route route, Exchange exchange) {
		// 参考 CamelInternalProcessor.InstrumentationAdvice 内部类
		final StopWatch answer = new StopWatch();
		// ====================== 这里要使用栈进行存储, 至于原因:
		// === 因为可能存在支线route, 例如 from().to("direct:500").to(xxxx) , 这里的"direct:500"会导致重建exchange, 也就是这里会再次被调用
		if(exchange.getProperty(EXCHANGE_PROERTIES_KEY_STOPWATCH_FOR_METRIC) == null){
			exchange.setProperty(EXCHANGE_PROERTIES_KEY_STOPWATCH_FOR_METRIC, new ArrayDeque<StopWatch>());
		}
		@SuppressWarnings("unchecked")
		Deque<StopWatch> deque = exchange.getProperty(EXCHANGE_PROERTIES_KEY_STOPWATCH_FOR_METRIC, Deque.class);
		deque.push(answer);
	}

	@Override
	public void onExchangeDone(Route route, Exchange exchange) {
		@SuppressWarnings("unchecked")
		Deque<StopWatch> deque  = exchange.getProperty(EXCHANGE_PROERTIES_KEY_STOPWATCH_FOR_METRIC, Deque.class);
		if(deque.size() > 1){
			// 主Route还没走完
			StopWatch subSW = deque.poll();
			Console.log("支exchange {} 耗時 {} millis; routeId [ {} ]", exchange.getExchangeId(), subSW.taken(), route.getId());
			return;
		}
    Console.log("主exchange {} 耗時 {} millis; routeId [ {} ]", exchange.getExchangeId(), 		           接口及      deque.poll().taken(), route.getId());
		}
	
	}

}
```


​			
​			



	// ====================== 配置
	@Component
	public class RoutePolicySample extends RouteBuilder {
	@Override
	public void configure() throws Exception {
	    // 全局配置, 避免需要在每个Route定义上添加
		getContext().addRoutePolicyFactory(new RoutePolicyFactoryInstrumentationImpl());
	
		from("rest:get,post:rp")//
		      .routeId("RoutePolicy") //
		      // 针对单个Route进行配置
		      // .routePolicy(new RoutePolicyFactoryInstrumentationImpl.RoutePolicyInstrumentationImpl())//
		      .process(customProcessor)//
		      .to("direct:500")//
		      .to("stream:err");
	
		from("direct:500") //
		      .to("stream:out");
		
	}
	}

## Camel MDC日志

```
@Bean
public CamelContextConfiguration contextConfiguration() {
    return new CamelContextConfiguration() {
        @Override
        public void beforeApplicationStart(CamelContext context) {
            context.setUseMDCLogging(true);
            context.setUnitOfWorkFactory(MyUnitOfWork::new);
        }

        @Override
        public void afterApplicationStart(CamelContext camelContext) {
        }
    };
}
Then, create your custom unit of work class

public class MyUnitOfWork extends MDCUnitOfWork {
    public MyUnitOfWork(Exchange exchange) {
        super(exchange);
        if( exchange.getProperty("myProp") != null){
            MDC.put("myProp", (String) exchange.getProperty("myProp"));
        }
    }
}



最后使用如下方式来进行日志打印：%X{myProp}



向线程池中传递MDC
@Bean
public Executor asyncServiceExecutor() {
    logger.info("start asyncServiceExecutor");
    // 使用VisiableThreadPoolTaskExecutor
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    // 配置核心线程数
    executor.setCorePoolSize(20);
    // 配置最大线程数
    executor.setMaxPoolSize(100);
    // 配置队列大小
    executor.setQueueCapacity(99999);
    // 配置线程池中的线程的名称前缀
    executor.setThreadNamePrefix("async-service-");

    // rejection-policy：当pool已经达到max size的时候，如何处理新任务
    // CALLER_RUNS：不在新线程中执行任务，而是有调用者所在的线程来执行
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    // 填充装饰器
    executor.setTaskDecorator(new MdcTaskDecorator());
    // 执行初始化
    executor.initialize();
    return TtlExecutors.getTtlExecutor(executor);
}

class MdcTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        Map<String, String> contextMap = MDC.getCopyOfContextMap();
        return () -> {
            try {
                if (contextMap != null) {
                    MDC.setContextMap(contextMap);
                }
                runnable.run();
            } finally {
                MDC.clear();
            }
        };
    }
}
```

