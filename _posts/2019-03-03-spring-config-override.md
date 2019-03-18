---
layout: post
title: 重写 spring boot 配置属性
categories: spring
description: 重写 spring boot 中 bean 的属性，适用于配置的加解密场景
keywords: spring boot
---
# 重写 spring boot 中的配置属性

## 背景
Spring boot 项目
因为安全上的要求，应用的一些敏感数据(诸如各种 token、key 之类)不能明文放在代码及其配置文件中。    

## 方案一：敏感信息加密
不让将明文放在代码或者配置文件中，那我直接将敏感信息加密之后再放配置文件，用的时候再解密。

### 优点
1. 思路简单，第一个想到的办法
2. 改造成本不高，用到密文的地方再调用一次解密方法就好

### 缺点
1. 代码侵入性强
2. 可以解决自定义类的问题，但是对于注入 spring.database.password 这类内置类就没有办法了

## 方案二：敏感信息存配置中心
敏感信息不能明文放在代码或者配置文件，那就放走配置中心嘛，反正公司有使用 spring cloud config配置中心。  
将敏感数据放在配置中心，等到运行时去拉取。只要配置中心数据库（配置中心持久端使用的是数据库）不泄漏，就一切搞定。

### 优点
1. 满足需求
2. 最简单的，只要梳理现有代码中的配置信息，统一放到配置中心即可
3. 绝大多数配置中心都有 Restful 接口，本方案连其他编程语言的情况都搞定了

### 缺点
1. 依赖配置中心，未接入的系统有接入成本
2. 敏感数据明文存储，还是不安全
3. 开发申请相关信息的时候，拿到的是明文，风险较大

## 方案三：配置中心加密存储
话都说到这个份上，那就还是用加密吧。  
好在 spring cloud config 本身也支持加密，具体使用方式建议自行查阅官方文档。
配置中心对于敏感数据加密存储，然后在客户端拉取配置时，解密后发给客户端。

### 优点
1. 基于方案二，只需要改配置中心服务端，仍然很简单
2. 实现了敏感数据的加密存储，开发看到的也是密文
3. 现有组件支持加解密，改造方便

### 缺点
1. 依赖配置中心，未接入的系统有接入成本
2. 服务端每次负责拉取配置，还要负责解密，性能成本比较高
3. 如果解密方法依赖项目相关信息，配置中心系统复杂度急剧升高
4. 服务拉到的配置依然是明文，对于spring 配置中心，伪造请求获取敏感数据的成本比较低

## 方案四：重写配置中心客户端，在客户端解密
我们可以参考 spring cloud config 中的 *ConfigServicePropertySourceLocator* 方式自己实现 *PropertySourceLocator* 接口，在获取到远程配置之后，将拉到的配置解密后再放回上下文。

### 优点
1. 基于方案三，服务端存放加密信息，客户端负责解密
2. 敏感数据只能客户端才能解密，安全上基本达到要求
3. 服务端只关注配置，复杂度降低

### 缺点
依赖配置中心，未接入的系统有接入成本

## 方案五：ApplicationContextInitializer
既然配置中心和本地配置可以结合使用，那么spring 框架中肯定有一个地方是统一处理来自各地的配置，然后将配置注入到 bean 中。  
所以只要找到这个地方，赶在 spring 将属性注入之前，对其进行预处理，就可以实现配置的统一解密。
经过查找，就找到了 *ApplicationContextInitializer*。  
令人惊喜的是，spring 默认对它有一个实现 *EnvironmentDecryptApplicationInitializer*，我们看下它的注释:
```java
/**
 * 将 environment 中的属性解密，并给它们一个较高的优先级，让它们可以覆盖密文
 * Decrypt properties from the environment and insert them with high priority so they
 * override the encrypted values.
 *
 * @author Dave Syer
 *
 */
```

之前只知道spring cloud config server可以在服务端解密，原来 spring boot 本身就有对它的支持。  
那么接下来问题就简单了，只要看下它是怎么运行的，然后就能直接拿来用了。

```java
public class EnvironmentDecryptApplicationInitializer implements
		ApplicationContextInitializer<ConfigurableApplicationContext>, Ordered {

	private TextEncryptor encryptor;

	@Override
	public void initialize(ConfigurableApplicationContext applicationContext) {

		ConfigurableEnvironment environment = applicationContext.getEnvironment();
		MutablePropertySources propertySources = environment.getPropertySources();

		Set<String> found = new LinkedHashSet<>();
        // 这里拿到上下文的属性，开始执行解密
		Map<String, Object> map = decrypt(propertySources); 
		。。。。
    }

	private void decrypt(PropertySource<?> source, Map<String, Object> overrides) {
        EnumerablePropertySource<?> enumerable = (EnumerablePropertySource<?>) source;
        for (String key : enumerable.getPropertyNames()) {
            String value = source.getProperty(key).toString();
            // 这里很重要，如果数据以 {cipher} 开头，就证明其后的数据是需要解密的
            if (value.startsWith("{cipher}")) {
                value = value.substring("{cipher}".length());
                try {
                    value = this.encryptor.decrypt(value);
                    if (logger.isDebugEnabled()) {
                        logger.debug("Decrypted: key=" + key);
                    }
                }
                catch (Exception e) {
                    。。。
                }
            }
        }
	}
}

```
通过上面的代码，我们很容易发现：
1. *EnvironmentDecryptApplicationInitializer* 可以拿到上下文的 *environment*，里面有所有的配置信息
2. 遍历配置，找到值中以 *{cipher}* 开头的属性值
3. 截掉 *{cipher}* 之后，调用成员变量 *encryptor* 的 *decrypt(value)* 方法进行解密
4. 幸运的是 *TextEncryptor* 是一个接口，我们没有理由不重写它，尤其是它还是一个 bean！！！

```java
@Configuration
@ConditionalOnClass({ TextEncryptor.class })
@EnableConfigurationProperties(KeyProperties.class)
public class EncryptionBootstrapConfiguration {
    。。。
 
	@Autowired(required = false)
	private TextEncryptor encryptor;

	@Bean
	public EnvironmentDecryptApplicationInitializer environmentDecryptApplicationListener() {
		if (this.encryptor == null) {
			this.encryptor = new FailsafeTextEncryptor();
		}
		EnvironmentDecryptApplicationInitializer listener = new EnvironmentDecryptApplicationInitializer(
				this.encryptor);
		listener.setFailOnError(this.key.isFailOnError());
		return listener;
	}
}
```
通过上面的代码，我们知道 *EnvironmentDecryptApplicationInitializer* 中的 *encryptor* 是一个 *TextEncryptor* 类型的 bean， 并在启动时去上下文中寻找有没有已经有个 bean，如果有就用它，如果没有，就给一个默认实现（其实默认就是直接抛异常）  
**注意**
> 上面说的这几个类不是普通的 bean，它们处理的都是 spring 上下文信息，所以肯定不能是一个简单的 **@Service** 就能生效的。  
> 你需要通过 spring.factories 告诉 spring，这几个类需要在启动时第一时间加载。

```java
resources/META-INF/spring.factories:
org.springframework.cloud.bootstrap.BootstrapConfiguration=com.bishion.MyConfigureConfiguration

@Configuration
public class MyConfigureConfiguration {
    @Bean
    public TextEncryptor textEncryptor(){
        return new MyTextEncryptor();
    }

}

public class MyTextEncryptor implements TextEncryptor {
    @Override
    public String encrypt(String s) {
        // 这个是加密方法
        return "xxx";
    }

    @Override
    public String decrypt(String s) {
        // 这个是解密方法
        return "bishion";
    }
}
```
### 优点
1. 基于方案四，业务系统不用关心解密实现
2. 业务系统可以摆脱对配置中心依赖

### 缺点
密文前面必须配置 **{cipher}**，而且该前缀 spring 处理得并不优雅，并没有给客户端太多的定制自由。该凭空多出来的前缀，会给接入方带来困惑：
1. 不知道为何会有这个前缀
2. yml 格式要求这种情况需要加单引号配置： com.bishion.key: '{cipher}密文'

## 方案六： 自定义 ApplicationContextInitializer
鉴于方案五的一些缺点，我们可以参考 *EnvironmentDecryptApplicationInitializer* 的实现思路，自己实现一套解密逻辑，包括自定义逻辑，自定义解密方法等功能。  
方法很简单：
1. 实现 *ApplicationContextInitializer* 接口
2. 实现 *TextEncryptor* 接口
3. 别忘了在 spring.factories 中加上启动配置

```java
resources/META-INF/spring.factories:
org.springframework.cloud.bootstrap.BootstrapConfiguration=com.bishion.MyConfigureConfiguration

@Configuration
public class MyConfigureConfiguration {
    @Bean
    public TextEncryptor textEncryptor(){
        return new MyTextEncryptor();
    }
    @Bean
    public MyDecryptApplicationInitializer myDecryptApplicationListener(TextEncryptor encryptor) {
        MyDecryptApplicationInitializer listener = new MyDecryptApplicationInitializer(
                encryptor);
        return listener;
    }
}
public class MyDecryptApplicationInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext>{

    private TextEncryptor encryptor;
    public MyDecryptApplicationInitializer(TextEncryptor encryptor) {
        this.encryptor = encryptor;
    }
    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        。。。
        Map<String, Object> map = decrypt(propertySources);
    }
    
    public Map<String, Object> decrypt(PropertySources propertySources) {
        Map<String, Object> overrides = new LinkedHashMap<>();
        // 解密逻辑
        return overrides;
    }
}
```

### 优点
1. 基于方案五，高度定制化，功能基本全部满足
2. 客户端接入简单，只需要引入相关 jar 包即可，无需侵入现有代码
3. 摆脱对配置中心的依赖

### 缺点
又定制了一个组件，维护成本增加。

## 方案七：jasypt-spring-boot
java 最让人无奈的地方在于：很多时候，你为了解决一个场景抓狂好多天才找到完美解决方案，甚至已经熬夜写完代码，然后洋洋得意地准备发个博客吹嘘一番的时候，突然发现，github 跳出了一大堆类似组件。心塞！  
当然我不是最惨的，在找到第五种方案的时候就发现了这个组件：jasypt-spring-boot  
说不心痛是不可能的，所以我已经没力气研究它了，感兴趣的同学移步至下方链接，自行查阅。  
https://github.com/ulisesbocchio/jasypt-spring-boot

### 优点
1. 方案六的所有优点
2. 看提交记录，该组件还是比较活跃的，对spring boot 2.0 都已经支持了。

### 缺点
不是我写的

## 思考
是在拿到需求之后，随着对需求和功能的不断深入理解，才逐渐得到了上述七种解决方案。  
最好解决方案并不是一蹴而就的，它是一个反复磨合和权衡的过程。  
好吧，我最想说的是，面向 github 编程才是正经