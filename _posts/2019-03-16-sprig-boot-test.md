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

源码地址：https://github.com/bishion/springboot-test.git

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
        BDDMockito.given(userService.queryUser(null)).willReturn(Collections.emptyList());
        List<User> userList = new ArrayList<>(1);
        User user = new User();
        user.setUsername("bishion");
        userList.add(user);
        BDDMockito.given(userService.queryUser(BDDMockito.startsWith("bi"))).willReturn(userList);

        this.mvc.perform(MockMvcRequestBuilders.get("/query")
                .param("username", "bishion"))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.jsonPath("$.length()", "").value(1));
        this.mvc.perform(MockMvcRequestBuilders.get("/query")
                .param("username", "guo"))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.jsonPath("$.length()", "").isNumber());

    }
}
```
### 注意
1. UserController 依赖了 UserService，因此要使用 **@MockBean** 对该 bean 做 mock
2. UserController.query() 调用了 UserService.query() 方法，因此要使用 **BDDMockito.given().willReturn()** 对该方法做mock
3. **MockMvcResultMatchers** 中使用了 jsonPath，可以对返回值做解析

### 场景三：Service 的单测

