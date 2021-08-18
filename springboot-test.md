# Springboot Test

```java
@RunWith(SpringRunner.class)
@SpringBootTest //告诉SpringBoot去寻找主配置类(例如使用@SpringBootApplication注解标注的类)，并使用它来启动一个Spring application context;
public class UserController01Test {

    @Autowired
    private UserController userController;

    //测试@SpringBootTest是否会将@Component加载到Spring application context
    @Test
    public void testContexLoads(){
        Assert.assertThat(userController,notNullValue());
    }

}
```

Spring Test支持的一个很好的特性是应用程序上下文在测试之间缓存，因此如果在测试用例中有多个方法，或者具有相同配置的多个测试用例，它们只会产生启动应用程序一次的成本。使用@DirtiesContext注解可以清空缓存，让程序重新加载。



真正的启动服进行测试 

```java
@RunWith(SpringRunner.class)
//SpringBootTest.WebEnvironment.RANDOM_PORT设置随机端口启动服务器（有助于避免测试环境中的冲突）
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT) 
public class HttpRequestTest {

    //使用@LocalServerPort将端口注入进来
    @LocalServerPort
    private int port;

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    public void greetingShouldReturnDefaultMessage() throws Exception {
        Assert.assertThat(this.restTemplate.getForObject("http://localhost:" + port + "/",String.class),
                Matchers.containsString("Hello World"));
    }
}
```

使用@AutoConfigureMockMvc注解自动注入MockMvc

```java
@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc //不启动服务器,使用mockMvc进行测试http请求。启动了完整的Spring应用程序上下文，但没有启动服务器
public class UserController02Test {

    @Autowired
    private MockMvc mockMvc;

    /**
     * .perform() : 执行一个MockMvcRequestBuilders的请求；MockMvcRequestBuilders有.get()、.post()、.put()、.delete()等请求。
     * .andDo() : 添加一个MockMvcResultHandlers结果处理器,可以用于打印结果输出(MockMvcResultHandlers.print())。
     * .andExpect : 添加MockMvcResultMatchers验证规则，验证执行结果是否正确。
     */
    @Test
    public void shouldReturnDefaultMessage() throws Exception {
        this.mockMvc.perform(get("/"))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(content().string(containsString("Hello World")));
    }

}
```

使用@WebMvcTest只初始化Controller层

```java
@RunWith(SpringRunner.class)
//使用@WebMvcTest只实例化Web层，而不是整个上下文。在具有多个Controller的应用程序中，
// 甚至可以要求仅使用一个实例化，例如@WebMvcTest(UserController.class)
@WebMvcTest(UserController.class)
public class UserController03Test {

    @Autowired
    private MockMvc mockMvc;

    @Test
    public void shouldReturnDefaultMessage() throws Exception {
        this.mockMvc.perform(get("/"))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(content().string(containsString("Hello World")));
    }

}
```

使用mockito框架的@MockBean注解进行模拟

```java
@RunWith(SpringRunner.class)
//使用@WebMvcTest只实例化Web层，而不是整个上下文。在具有多个Controller的应用程序中，
// 甚至可以要求仅使用一个实例化，例如@WebMvcTest(UserController.class)
@WebMvcTest(UserController.class)
public class UserController03Test {

    @Autowired
    private MockMvc mockMvc;

    //模拟出一个userService
    @MockBean
    private UserService userService;

    @Test
    public void greetingShouldReturnMessageFromService() throws Exception {
        //模拟userService.findByUserId(1)的行为
        when(userService.findByUserId(1)).thenReturn(new User(1,"张三"));

        String result = this.mockMvc.perform(get("/user/1"))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(content().contentType(MediaType.APPLICATION_JSON_UTF8))
                .andExpect(jsonPath("$.name").value("张三"))
                .andReturn().getResponse().getContentAsString();

        System.out.println("result : " + result);
    }

}
```