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

## 一次动态生成webservice记录

### Json to Pojo

```java
import com.google.gson.*;
import com.sun.codemodel.*;

import javax.xml.bind.annotation.XmlAccessType;
import javax.xml.bind.annotation.XmlAccessorType;
import javax.xml.bind.annotation.XmlElement;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.OutputStream;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

/**
 * @author mipengcheng3
 * Parse a json object to *.java file, the *.java file will be generated recursively
 */
public final class JsonToPojo {

    private final List<String> classes = new ArrayList<>();

    private final JCodeModel codeModel = new JCodeModel();

    public List<String> getClasses() {
        return classes;
    }

    public void loadJson(String json, String classPackage) {
        JsonElement root = JsonParser.parseString(json);
        generateCode(root, classPackage);
    }

    void generateCode(JsonElement root, String classPackage) {

        int lastIndexDot = classPackage.lastIndexOf(".");
        String packageName = classPackage.substring(0, lastIndexDot);
        String className = classPackage.substring(lastIndexDot + 1, classPackage.length());

        generateClass(packageName, className, root);

        try {
            CodeWriter codeWriter = new CodeWriter() {
                List<ByteArrayOutputStream> bytes = new ArrayList<>();
                @Override
                public OutputStream openBinary(JPackage pkg, String fileName) throws IOException {
                    ByteArrayOutputStream out = new ByteArrayOutputStream();
                    bytes.add(out);
                    return out;
                }
                @Override
                public void close() throws IOException {
                    if (null != bytes) {
                        for (ByteArrayOutputStream aByte : bytes) {
                            classes.add(new String(aByte.toByteArray()));
                            aByte.close();
                        }
                        bytes = null;
                    }

                }
            };

            codeModel.build(codeWriter);

        } catch (Exception e) {
            throw new IllegalStateException("Couldn't generate Pojo", e);
        }
    }

    private JClass generateClass(String packageName, String className, JsonElement jsonElement) {

        JClass elementClass = null;

        if (jsonElement.isJsonNull()) {
            elementClass = codeModel.ref(Object.class);

        } else if (jsonElement.isJsonPrimitive()) {

            JsonPrimitive jsonPrimitive = jsonElement.getAsJsonPrimitive();
            elementClass = getClassForPrimitive(jsonPrimitive);

        } else if (jsonElement.isJsonArray()) {

            JsonArray array = jsonElement.getAsJsonArray();
            elementClass = getClassForArray(packageName, className, array);

        } else if (jsonElement.isJsonObject()) {

            JsonObject jsonObj = jsonElement.getAsJsonObject();
            elementClass = getClassForObject(packageName, className, jsonObj);
        }

        if (elementClass != null) {
            return elementClass;
        }

        throw new IllegalStateException("jsonElement type not supported");
    }

    private JClass getClassForObject(String packageName, String className, JsonObject jsonObj) {

        Map<String, JClass> fields = new LinkedHashMap<String, JClass>();

        for (Map.Entry<String, JsonElement> element : jsonObj.entrySet()) {

            String fieldName = element.getKey();
            String fieldUppercase = getFirstUppercase(fieldName);

            JClass elementClass = generateClass(packageName, fieldUppercase, element.getValue());
            fields.put(fieldName, elementClass);
        }

        String classPackage = packageName + "." + className;
        generatePojo(classPackage, fields);

        JClass jclass = codeModel.ref(classPackage);
        return jclass;
    }

    private JClass getClassForArray(String packageName, String className, JsonArray array) {

        JClass narrowClass = codeModel.ref(Object.class);
        if (array.size() > 0) {
            String elementName = className;
            if (className.endsWith("ies")) {
                elementName = elementName.substring(0, elementName.length() - 3) + "y";
            } else if (elementName.endsWith("s")) {
                elementName = elementName.substring(0, elementName.length() - 1);
            }

            narrowClass = generateClass(packageName, elementName, array.get(0));
        }

        String narrowName = narrowClass.name();
        Class<?> boxedClass = null;
        if (narrowName.equals("int")) {
            boxedClass = Integer.class;
        } else if (narrowName.equals("long")) {
            boxedClass = Long.class;
        } else if (narrowName.equals("double")) {
            boxedClass = Double.class;
        }
        if (boxedClass != null) {
            narrowClass = codeModel.ref(boxedClass);
        }

        JClass listClass = codeModel.ref(List.class).narrow(narrowClass);
        return listClass;
    }

    private JClass getClassForPrimitive(JsonPrimitive jsonPrimitive) {

        JClass jClass = null;

        if (jsonPrimitive.isNumber()) {
            Number number = jsonPrimitive.getAsNumber();
            double doubleValue = number.doubleValue();

            if (doubleValue != Math.round(doubleValue)) {
                jClass = codeModel.ref("double");
            } else {
                long longValue = number.longValue();
                if (longValue >= Integer.MAX_VALUE) {
                    jClass = codeModel.ref("long");
                } else {
                    jClass = codeModel.ref("int");
                }
            }
        } else if (jsonPrimitive.isBoolean()) {
            jClass = codeModel.ref("boolean");
        } else {
            jClass = codeModel.ref(String.class);
        }
        return jClass;
    }
    public void generatePojo(String className, Map<String, JClass> fields) {

        try {
            JDefinedClass definedClass = codeModel._class(className);
            JAnnotationUse annotate = definedClass.annotate(XmlAccessorType.class);
            annotate.param("value", XmlAccessType.NONE);

            for (Map.Entry<String, JClass> field : fields.entrySet()) {

                addGetterSetter(definedClass, field.getKey(), field.getValue());
            }

        } catch (Exception e) {
            throw new IllegalStateException("Couldn't generate Pojo", e);
        }
    }

    private void addGetterSetter(JDefinedClass definedClass, String fieldName, JClass fieldType) {

        String fieldNameWithFirstLetterToUpperCase = getFirstUppercase(fieldName);

        JFieldVar field = definedClass.field(JMod.PRIVATE, fieldType, fieldName);
        field.annotate(XmlElement.class);

        String getterPrefix = "get";
        String fieldTypeName = fieldType.fullName();
        if (fieldTypeName.equals("boolean") || fieldTypeName.equals("java.lang.Boolean")) {
            getterPrefix = "is";
        }
        String getterMethodName = getterPrefix + fieldNameWithFirstLetterToUpperCase;
        JMethod getterMethod = definedClass.method(JMod.PUBLIC, fieldType, getterMethodName);
        JBlock block = getterMethod.body();
        block._return(field);

        String setterMethodName = "set" + fieldNameWithFirstLetterToUpperCase;
        JMethod setterMethod = definedClass.method(JMod.PUBLIC, Void.TYPE, setterMethodName);
        String setterParameter = fieldName;
        setterMethod.param(fieldType, setterParameter);
        setterMethod.body().assign(JExpr._this().ref(fieldName), JExpr.ref(setterParameter));
    }

    private String getFirstUppercase(String word) {

        String firstLetterToUpperCase = word.substring(0, 1).toUpperCase();
        if (word.length() > 1) {
            firstLetterToUpperCase += word.substring(1, word.length());
        }
        return firstLetterToUpperCase;
    }
}
```

### 使用groovy编译器动态加载类，并暴露webservice

```java
    protected void doStart() throws Exception {
        JsonToPojo jsonToPojo = new JsonToPojo();
        String serviceName = getEndpoint().getServiceName();
        int dotIndex = serviceName.lastIndexOf(".");
        String packageName = serviceName.substring(0, dotIndex);
        String serviceClassName = serviceName.substring(dotIndex+1);

        jsonToPojo.loadJson(getEndpoint().getLoader().load(getEndpoint().getParamJsonKey()), packageName + ".Request");
        jsonToPojo.loadJson(getEndpoint().getLoader().load(getEndpoint().getReturnJsonKey()), packageName + ".Response");

        //编译
        GroovyClassLoader groovyClassLoader = new GroovyClassLoader();
        for (int i = 0; i < jsonToPojo.getClasses().size(); i++) {
            String s = jsonToPojo.getClasses().get(i);
            GroovyCodeSource gcs = new GroovyCodeSource(s, i + ".groovy", "/groovy/script");
            gcs.setCachable(true);
            groovyClassLoader.parseClass(s);
        }

        final String serviceClass = "package "+ packageName +";\n" +
                "\n" +
                "import javax.jws.WebService;\n" +
                "\n" +
                "@WebService\n" +
                "public interface "+ serviceClassName +" {\n" +
                "    Response handle(Request request);\n" +
                "}";

        Class serviceClazz = groovyClassLoader.parseClass(serviceClass);


        final String implClass = "package "+ packageName+";\n" +
                "\n" +
                "import com.alibaba.fastjson.JSONObject;\n" +
                "import com.jd.jnos.baize.component.webservice.BaizeWebServiceConsumer;\n" +
                "import org.apache.camel.Exchange;\n" +
                "import org.apache.camel.ExchangePattern;\n" +
                "\n" +
                "public class "+serviceClassName+"Impl implements "+ serviceClassName +"{\n" +
                "    private BaizeWebServiceConsumer baizeWebServiceConsumer;\n" +
                "\n" +
                "    public "+serviceClassName+"Impl(BaizeWebServiceConsumer baizeWebServiceConsumer) {\n" +
                "        this.baizeWebServiceConsumer = baizeWebServiceConsumer;\n" +
                "    }\n" +
                "\n" +
                "    public Response handle(Request request) {\n" +
                "        Exchange exchange = BaizeWebServiceConsumer.ReadSoapHeader.EXCHANGE_THREAD_LOCAL.get();\n" +
                "        if (exchange == null) {\n" +
                "            throw new IllegalStateException(\"exchange missing\");\n" +
                "        }\n" +
                "        String s = JSONObject.toJSONString(request);\n" +
                "        exchange.getIn().setBody(s);\n" +
                "        exchange.setPattern(ExchangePattern.InOut);\n" +
                "        try {\n" +
                "            baizeWebServiceConsumer.getProcessor().process(exchange);\n" +
                "        } catch (Exception e) {\n" +
                "            exchange.setException(e);\n" +
                "        }\n" +
                "        if (exchange.getException() != null) {\n" +
                "            baizeWebServiceConsumer.getExceptionHandler().handleException(\"Error processing exchange\", exchange, exchange.getException());\n" +
                "        } else {\n" +
                "            Object body = exchange.getMessage().getBody();\n" +
                "            if (body instanceof String) {\n" +
                "                String strBody = (String)body;\n" +
                "                Response response = JSONObject.parseObject(strBody, Response.class);\n" +
                "                return response;\n" +
                "            }\n" +
                "        }\n" +
                "        return new Response();\n" +
                "    }\n" +
                "}";

        Class implClazz = groovyClassLoader.parseClass(implClass);

        ServerFactoryBean svrFactory = new ServerFactoryBean();
        svrFactory.setServiceClass(serviceClazz);
        svrFactory.setAddress(serviceName.toLowerCase().replaceAll("\\.", "/"));
        svrFactory.getInInterceptors().add(new ReadSoapHeader(this));


        try {
            svrFactory.setServiceBean(implClazz.getDeclaredConstructor(BaizeWebServiceConsumer.class).newInstance(this));
        } catch (Exception e) {
            e.printStackTrace();
        }
        if (this.server == null) {
            this.server = svrFactory.create();
            super.doStart();
            log.info("{} 启动成功", serviceName);
        }
    }
```

### 通过Servlet的方式来暴露接口

```java
@Configuration
public class CxfServletConfig {

    @Bean
    public SpringBus cxf() {
        return new SpringBus();

    }

    @Bean
    public ServletRegistrationBean<CXFServlet> cxfServlet (SpringBus cxf) throws NoSuchMethodException {
        CXFServlet cxfServlet = new CXFServlet();
        cxfServlet.setBus(cxf);
        BusFactory.setDefaultBus(cxfServlet.getBus());
        return new ServletRegistrationBean<>(cxfServlet, "/webservice/*");
    }


    @Bean
    public ComponentCustomizer webServiceComponentCustomizer(SpringBus bus) {
        return new ComponentCustomizer() {
            @Override
            public void configure(String name, Component target) {
                BaizeWebServiceComponent baizeWebServiceComponent = BaizeWebServiceComponent.class.cast(target);
                baizeWebServiceComponent.setBus(bus);
                baizeWebServiceComponent.setLoader(new JsonParamLoader() {
                    @Override
                    public String load(String key) {
                        if (key.equalsIgnoreCase("1")) {
                            return "{\"extensions\":[{\"value\":\"1\",\"key\":\"1\"},{\"value\":\"2\",\"key\":\"2\"}],\"data\":{\"srcSysCode\":9223372036854775807,\"srcTenantId\":\"1\",\"srcParentId\":\"1\",\"srcId\":\"1\",\"code\":\"c\",\"name\":\"name\",\"desc\":\"desc\",\"status\":\"s\",\"srcModified\":9223372036854775807,\"srcCreated\":9223372036854775807},\"tenantToken\":\"token\"}";
                        } else {
                            return "{\"jingdong_ierp_saas_transformer_TransformerProductApi_addOrUpdateBrand_response\":{\"transformerId\":\"1\"}}";
                        }
                    }
                });
            }

            @Override
            public boolean isEnabled(String name, Component target) {
                return target instanceof BaizeWebServiceComponent;
            }
        };
    }
}
```

### camel以外的相关依赖

```xml
        <dependency>
            <groupId>org.apache.cxf</groupId>
            <artifactId>cxf-core</artifactId>
            <version>3.5.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.cxf</groupId>
            <artifactId>cxf-rt-frontend-jaxws</artifactId>
            <version>3.5.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.cxf</groupId>
            <artifactId>cxf-rt-transports-http</artifactId>
            <version>3.5.2</version>
        </dependency>

        <dependency>
            <groupId>javax.xml.ws</groupId>
            <artifactId>jaxws-api</artifactId>
            <version>2.3.1</version>
        </dependency>
        <dependency>
            <groupId>com.sun.xml.ws</groupId>
            <artifactId>jaxws-rt</artifactId>
            <version>2.1.3</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/javax.jws/javax.jws-api -->
        <dependency>
            <groupId>javax.jws</groupId>
            <artifactId>javax.jws-api</artifactId>
            <version>1.1</version>
        </dependency>
        <dependency>
            <groupId>com.sun.xml.messaging.saaj</groupId>
            <artifactId>saaj-impl</artifactId>
            <version>1.5.1</version>
        </dependency>
        <dependency>
            <groupId>com.sun.codemodel</groupId>
            <artifactId>codemodel</artifactId>
            <version>2.6</version>
        </dependency>
        <dependency>
            <groupId>com.google.code.gson</groupId>
            <artifactId>gson</artifactId>
            <version>2.9.0</version>
        </dependency>
        <dependency>
            <groupId>jdom</groupId>
            <artifactId>jdom</artifactId>
            <version>1.1</version>
        </dependency>

        <dependency>
            <groupId>org.codehaus.groovy</groupId>
            <artifactId>groovy-all</artifactId>
            <version>2.4.13</version>
        </dependency>
    </dependencies>
```

