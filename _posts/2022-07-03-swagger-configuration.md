---
layout: post
title: 让 knife4j 配置更优雅
categories: swagger, spring, knife4j
description: 通过自定义配置, 让 knife4j 更简单
keywords: spring, swagger, knife4j
---

# 背景

项目使用 knife4j + swagger 作为文档管理工具, 在日常使用中, 发现一些不便之处:
1. swagger 的配置属于静态信息, 跟系统配置放在一起, 不易维护
2. knife4j 开箱即用, 默认开启, 生产环境有安全风险. 如果需要关闭, 需要显式指定配置：
```yaml
knife4j:
  enable: true
  production: true
```

# 解决方式

## 使用 *@PropertySource* 手动引入自定义配置文件
```java
@PropertySource(value = "classpath:swagger.properties")
@EnableConfigurationProperties(SwaggerProperties.class)
@EnableSwagger2
public class SwaggerConfiguration {
    .....
}
```

因为 properties 文件不支持中文, 可读性太差, 这里我们可以为它自定义解析器

```java
public class YamlPropertySourceFactory implements PropertySourceFactory {
    @Override
    public PropertySource<?> createPropertySource(String s, EncodedResource encodedResource) throws IOException {
        YamlPropertiesFactoryBean factoryBean = new YamlPropertiesFactoryBean();

        factoryBean.setResources(encodedResource.getResource());
        factoryBean.afterPropertiesSet();
        Properties properties =  factoryBean.getObject();
        String sourceName = s != null ? s : encodedResource.getResource().getFilename();
        return new PropertiesPropertySource(sourceName, properties);
    }
}
```

使用方式
```java
@PropertySource(value = "classpath:swagger.yml", factory = YamlPropertySourceFactory.class)
@EnableConfigurationProperties(SwaggerProperties.class)
@EnableSwagger2
public class SwaggerConfiguration {
    .....
}
```

## 实现 *EnvironmentPostProcessor* 手动注入配置

```java
public class Knife4jEnvPostProcessor implements EnvironmentPostProcessor {
    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
        Properties properties = new Properties();
        properties.put("knife4j.enable", true);
        // 这里可以根据自己项目需求，自定义生产环境判断逻辑
        properties.put("knife4j.production", EnvUtil.ENV_IS_PRD);

        environment.getPropertySources().addFirst(new PropertiesPropertySource("swaggerConfig", properties));
    }
}
```

需要在 /META-INF/spring.factories 指定配置, 以便在启动时加载该处理器
```property
org.springframework.boot.env.EnvironmentPostProcessor=*.Knife4jEnvPostProcessor
```

# 注意
因配置的加载优先于 bean 的初始化, 所以无法在 *Knife4jEnvPostProcessor* 中注入 bean.