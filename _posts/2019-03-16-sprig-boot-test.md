---
layout: post
title: spring boot 单测
categories: spring
description: spring boot 常见单测方式介绍
keywords: spring boot
---
# spring boot 项目常见单测方式

## 背景
最近组里在推单测，突然想起自己已经三十多年不写单测了，不由心生惭愧。  
默默打开 spring boot 官网，看了看相关单测的文档，整理了下自己的需要的一些单测场景。  
由于内容比较基础，只管怎么做，不问为什么，出于节省时间的目的，建议写过单测的同学可以不用往下看了。

## 代码环境
- spring cloud Edgware.SR5 版本
- 使用 h2 内置数据库
- 持久层使用 mybatis 1.3.3
- spring cloud 组件只使用了 spring cloud config
- 依赖 spring-boot-starter-test 组件
- 依赖 mybatis-spring-boot-starter-test 组件

## 场景一：简单的 Controller
入门场景，测试一个Controller，它没有依赖，没有参数。  
我只想测试该 Controller，并不想将整个应用启动
```java
@RestController
public class HealthController {

    @RequestMapping("/health")
    public String health(){
        return "UP";
    }
}
```

测试代码：  
```java
@RunWith(SpringRunner.class)
@WebMvcTest(HealthController.class)
public class HealthControllerTest {
    @Autowired
    private MockMvc mvc;

    @Test
    public void health() throws Exception {
        this.mvc.perform(MockMvcRequestBuilders.get("/health").accept(MediaType.TEXT_PLAIN))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.content().string("UP"));
    }
}
```
### 注意
1. SpringRunner除了比 SpringJUnit4ClassRunner短一点，其他二者相同
2. 如果只想测试 Controller，那么注解中加上@WebMvcTest,并引入要测试的 Controller 即可
3. 自动注入 **MockMvc**，为什么不用 RestTemplate，因为此时应用并没有真正启动，相关端口也没有打开
4. **MockMvcRequestBuilders** 用来构建 http 请求参数，它能做的有很多，这里不展开
5. **MockMvcResultMatchers** 用来做请求结果的数据解析，这里也不展开

## 场景二：复杂的 Controller
上述示例太过简单，一个普通的 Controller 还应该包括如下元素：
- Service
- 请求参数
- 返回 Json数据
- 甚至某些字段是从 配置中心获取的  
- 
```java
@RestController
public class UserController {
    @Value("${config.username:bishion}")
    private String defaultUser;

    @Autowired
    private UserService userService;
     @RequestMapping("/query")
    public List<UserDTO> query(String username){
        if(StringUtils.isEmpty(userDTO.getUsername())){
            username = defaultUser;
        }
        List<User> users =  userService.queryUser(username);
        return transfer(users);    // DOList 转 DTOList
    }
}
```java
@RunWith(SpringRunner.class)
@WebMvcTest(UserController.class)
public class UserControllerTest {

    @Autowired
    private MockMvc mvc;

    @MockBean
    private UserService userService;

    @Test
    public void query() throws Exception {
        List<User> userList = new ArrayList<>(1);
        User user = new User();
        user.setUsername("bishion");
        userList.add(user);
        BDDMockito.given(userService.queryUser(null)).willReturn(Collections.emptyList());
        BDDMockito.given(userService.queryUser(BDDMockito.startsWith("bi"))).willReturn(userList);
        BDDMockito.given(userService.queryUser(BDDMockito.startsWith("${config.username}"))).willReturn(userList);

        this.mvc.perform(MockMvcRequestBuilders.get("/query")
                .param("username", "bishion"))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.jsonPath("$.length()", "").value(1));

        this.mvc.perform(MockMvcRequestBuilders.get("/query")
                .param("username", "guo"))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.jsonPath("$.length()", "").value(0));

        this.mvc.perform(MockMvcRequestBuilders.get("/query"))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.jsonPath("$.length()", "").value(1));
    }
}
```
### 注意
1. UserController 依赖了 UserService，因此要使用 **@MockBean** 对该 bean 做 mock
2. UserController.query() 调用了 UserService.query() 方法，因此要使用 **BDDMockito.given().willReturn()** 对该方法做mock
3. **MockMvcResultMatchers** 中使用了 jsonPath，可以对返回值做解析
4. 如果 Controller 依赖了外部配置而且没有配置值，那么它直接使用 **@Value** 注解的值，这里为 *${config.username}*
5. 如果你想为依赖的外部配置添加测试配置，可以使用 **@TestPropertySource** 比如：@TestPropertySource(properties = "config.username=test")

### 场景三：Service 的单测
需要对一个 Service 做单测，该 Service 调用了其它 Service
```java
@FeignClient(name = "remote-service",url = "https://www.baidu.com")
public interface BaiduService {
    @RequestMapping("/")
    String request();
}
@Service
public class CallRemoteService {
    @Autowired
    private BaiduService baiduService;

    public String callBaidu() {
        String baidu = baiduService.request();

        return baidu.substring(2, 4);
    }
}
```

测试代码：
```java
@RunWith(SpringRunner.class)
@Import(CallRemoteService.class)
public class CallRemoteServiceTest {

    @MockBean
    private BaiduService baiduService;

    @Autowired
    private CallRemoteService callRemoteService;

    @Test
    public void callBaidu() {
        BDDMockito.given(this.baiduService.request()).willReturn("SUCCESS");
        String result = callRemoteService.callBaidu();
        Assert.hasLength(result,"返回数据不应该为空");
        Assert.isTrue(result.length() == 2,"返回数据长度应为2");

    }
}
```
### 注意
1. 如果需要测试一个 Service, 需要使用 **@Import** 将该 Service 引入到上下文
2. 因为该 Service 调用了别的 Service，所以需要 **@MockBean**

## 场景四：mybatis
针对 dao 层的单测：
1. 使用了 mybatis-spring-boot-starter 依赖
2. 使用了 注解和 XML 两种方式的 mapper

```java
@Mapper
public interface UserDao {

    @Insert("insert into User values(null,#{username},#{age})")
    @Options(useGeneratedKeys=true,keyColumn = "id")
    Integer addUser(User user);

    @Select("select * from User where username = #{username}")
    List<User> queryUserByName(String username);
    // 该方法为 xml 配置
    Integer updateUserById(User user);
}
```
测试代码：
```java
@RunWith(SpringRunner.class)
@MybatisTest
//@Rollback(false)  
//@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
public class UserDaoTest {

    @Autowired
    private UserDao userDao;

    @Test
    public void testUserDao(){
        // 测试插入数据
        User user = new User();
        user.setUsername("bizi");
        user.setAge(18);
        userDao.addUser(user);

        // 测试更新数据
        User newUser = new User();
        newUser.setId(user.getId());
        newUser.setUsername("bishion");
        userDao.updateUserById(newUser);

        // 测试查询数据
        List<User> userList = userDao.queryUserByName("bishion");
        Assert.notNull(user.getId(),"未获取到ID");
        Assert.notEmpty(userList,"未查到插入数据");
    }

}
```
### 注意：
1. **@AutoConfigureTestDatabase** 用于指定是否使用 mybatis 单测内置数据库，默认是使用
2. 该测试用例依赖 mybatis-spring-boot-starter-test
3. 因为 **兼容性** 问题，请不要将 **@MapperScan** 注解放到Application启动类上，否则会报错
4. 如果你使用的是内置数据库，需要在 src/test/resources 下面添加 schema.sql，里面放入建表语句

## 场景五：Feign 的测试
虽然说起来很傻，但是有时候还真的需要单独测试 Feign
```java
@FeignClient(name = "remote-service",url = "https://www.baidu.com")
public interface BaiduService {
    @RequestMapping("/")
    String request();
}
```
测试代码：

```java
@RunWith(SpringRunner.class)
@RestClientTest(BaiduService.class)
@ImportAutoConfiguration({RibbonAutoConfiguration.class, FeignRibbonClientAutoConfiguration.class, FeignAutoConfiguration.class})
public class BaiduServiceTest {

    @Autowired
    private BaiduService baiduService;

    @Test
    public void request(){
        Assert.hasText(baiduService.request(),"未查到数据");
    }
}
```
### 注意：
1. 因为测试 Feign 需要相关的上下文，所以要手动引入
2. 需要使用 **@RestClientTest** 将被测接口加进来
3. 个人觉得直接使用 RestTemplate 也挺好

## 场景六：集成测试
需要将整个项目启动，然后再单测
```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.ANY)
public class UserController {
    @Autowired
    private TestRestTemplate template;

    @Test
    public void testAddUser(){
        UserDTO userDTO = new UserDTO();
        userDTO.setUsername("bishion");
        String result = template.postForEntity("/addUser",userDTO, String.class).getBody();
        Assert.isTrue("SUCCESS".endsWith(result),"返回不成功"+result);
    }
    @Test
    public void testQueryUser(){
        MultiValueMap<String, String> map= new LinkedMultiValueMap<>();
        map.add("username", "bishion");
        List<UserDTO> result = template.postForEntity("/query",map, List.class).getBody();
        Assert.isTrue(result.size()>0,"没有查出数据");

    }
}
```
### 注意
1. 如果你之前设置了 schema.sql，这里一定要显式设置 **AutoConfigureTestDatabase** 为 Replace.ANY
2. 有了 **@SpringBootTest**，可以将 TestRestTemplate 自动注入

## 总结
1. 因为时间仓促，只是简单看了下文档做了个总结，所以很多问题没有深究
2. 这里只是列了一些典型场景，后续本文档会更新，放在 https://bishion.github.io/2019/03/16/sprig-boot-test/
3. 本文源码放在 https://github.com/bishion/springboot-test.git
3. spring boot 关于测试这一块的文档写的太抽象了