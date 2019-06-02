---
layout: post
title: spring cloud feign 添加headers
categories: spring, feign, headers
description: spring cloud feign 添加headers
keywords: springcloud, feign, openfeign, headers
---
# 为 springcloud feign 添加自定义headers

## 背景
最近在调用一个接口，接口要求将token放在header中传递。由于我的项目使用了feign, 那么给请求中添加 header 就必须要去feign中找方法了。

## 方案一：自定义 RequestInterceptor
在给 @FeignClient 注解的接口生成代理对象的时候，有这么一段：
```java
class FeignClientFactoryBean implements FactoryBean<Object>, InitializingBean, ApplicationContextAware {
    @Override
	public Object getObject() throws Exception {
		return getTarget();
	}
    // getTarget() 最终会调用到 configureUsingConfiguration()
    protected void configureUsingConfiguration(FeignContext context,Feign.Builder builder) {
		Map<String, RequestInterceptor> requestInterceptors = context.getInstances(this.contextId, RequestInterceptor.class);
		if (requestInterceptors != null) {
			builder.requestInterceptors(requestInterceptors.values());
		}
        ...
	}
```

生成代理类时，会使用到 spring 上下文的 *RequestInterceptor*， 而 @FeignClient 的代理类在执行的时候，会去使用该拦截器：
```java
final class SynchronousMethodHandler implements MethodHandler {
	Request targetRequest(RequestTemplate template) {
      for (RequestInterceptor interceptor : requestInterceptors) {
        interceptor.apply(template);
      }
      return target.apply(template);
  }
}
```

所以自定义自己的拦截器，然后注入到  spring 上下文中，这样就可以在请求的上下文中添加自定义的请求头：
```java
@Service
public class MyRequestInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        template.header("my-header","header");
    }
}
```

### 优点
实现简单，使用现有接口注入即可

### 缺点
操作的是全局的 RequestTemplate，比较难以根据不同的服务方提供不同的 header。  
虽然可以在 template 中根据 uri 来判断不同的服务提供方，然后添加对应的header，但是凭空多了很多配置信息，维护也比较困难。

## 方案二：在 @RequestMapping 注解中增加 header 信息
既然我们用到了 openfeign 框架，那我们找找 openfeign 官方是怎么解决的(https://github.com/OpenFeign/feign)：
```java
// openfeign 官方文档
public interface ContentService {
  @RequestLine("GET /api/documents/{contentType}")
  @Headers("Accept: {contentType}")
  String getDocumentByType(@Param("contentType") String type);
}
```

通过上述官方代码示例，我们可以发现，其实使用原生的 API 就可以满足我们的需求：
```java
@FeignClient(name = "feign",url = "127.0.0.1:8080")
public interface FeignTest {
    @RequestMapping(value = "/test")
    @Headers({"app: test-app","token: ${test-app.token}"})
    String test();
}
```
然而比较遗憾的是，@Headers 并没有生效，生成的RequestTemplate中，没有上述两个 Header 信息。  
跟踪代码，我们发现，ReflectFeign在生成远程服务的代理类的时候，会通过 *Contract* 接口准备数据。  
而*@Headers* 注解没有生效的原因是：官方的 Contract 没有生效：
```java
class FeignClientFactoryBean implements FactoryBean<Object>, InitializingBean, ApplicationContextAware {
    protected Feign.Builder feign(FeignContext context) {
		Feign.Builder builder = get(context, Feign.Builder.class)
				// required values
				.logger(logger)
				.encoder(get(context, Encoder.class))
				.decoder(get(context, Decoder.class))
				.contract(get(context, Contract.class));
                ...
	}

}
```
对于 springcloud-openfeign 来说，在创建 Feign 相关类的时候，使用的是容器中注入的 Contract：
```java
@Bean
@ConditionalOnMissingBean
public Contract feignContract(ConversionService feignConversionService) {
    return new SpringMvcContract(this.parameterProcessors, feignConversionService);
}

public class SpringMvcContract extends Contract.BaseContract implements ResourceLoaderAware {
    @Override
	public MethodMetadata parseAndValidateMetadata(Class<?> targetType, Method method) {
		....
        // 注意这里，它只取了 RequestMapping 注解
		RequestMapping classAnnotation = findMergedAnnotation(targetType, RequestMapping.class);
		....
		parseHeaders(md, method, classAnnotation);
		}
		return md;
	}
}
```
到这里我们就梳理出来整个事情的来龙去脉了：
1. openfeign 是支持给方法加上自定义 header 的，它用的是自己的注解 *@Headers*
2. springcloud-openfeign 使用了 openfeign的核心功能，但是关于 *@Headers* 的注解没有使用
3. springcloud 使用了自己的 *SpringMvcContract* 来处理请求的相关资源信息，里面只使用 *@RequestMapping* 注解
```

我们比较容易想到的是，既然 *@RequestMapping* 注解中有 headers 的属性，我们可以试一下
```java
@FeignClient(name = "server",url = "127.0.0.1:8080")
public interface FeignTest {
    @RequestMapping(value = "/test",headers = {"app=test-app","token=${test-app.token}"})
    String test();
}
```
亲测可用，这样我们就可以给特定的服务单独定制头信息啦。

### 优点
实现更加简单了，甚至都不用自己实现接口，只需要自己在相关注解中增加对应属性配置即可

### 缺点
虽然不用给全局的请求增加header，但是对于相同的服务方，却要在每个@RequestMapping注解中添加相同的header配置，会比较麻烦，能否添加全局的呢?

## 方案三：自定义 Contract
通过 SpringMvcContract 代码我们也很容易发现，对于类的注解，它只会处理 *RequestMapping*，其它也都忽略了。  
那么如果我们重新定义自己的 Contract，就可以随心所欲实现自己的想要的功能啦。
1. 方便起见，我们直接复用 openfeign 的 *@Header* 
2. 简单起见，我们直接继承 *SpringMvcContract*
3. 自定义自己的 Contract，然后注入到 spring 上下文中
```java
@Service
// 为了处理简单，我们直接继承 SpringMvcContract
public class MyContract extends SpringMvcContract {
    // 该属性是为了使用 springcloud config
    private ResourceLoader resourceLoader;

    @Override
    protected void processAnnotationOnClass(MethodMetadata data, Class<?> clz) {
        if (clz.getInterfaces().length == 0) {
            。。。 这里复用原有 SpringMvcContract 逻辑

        }
        // 以下是新加的逻辑(其实是使用的 openfeign 自带的 Contract.Default的逻辑)
        if (clz.isAnnotationPresent(Headers.class)) {
            String[] headersOnType = clz.getAnnotation(Headers.class).value();
            Map<String, Collection<String>> headers = toMap(headersOnType);
            headers.putAll(data.template().headers());
            data.template().headers(null); // to clear
            data.template().headers(headers);
        }
    }
    private Map<String, Collection<String>> toMap(String[] input) {
        Map<String, Collection<String>> result =
                new LinkedHashMap<String, Collection<String>>(input.length);
        for (String header : input) {
            。。。。这里使用的 openfeign 自带的 Contract.Default的逻辑，但是为了使用 springcloud config，又调用了 resolve方法
            result.get(name).add(resolve(header.substring(colon + 1).trim()));
        }
        return result;
    }
    private String resolve(String value) {
        if (StringUtils.hasText(value)
                && resourceLoader instanceof ConfigurableApplicationContext) {
            return ((ConfigurableApplicationContext) this.resourceLoader).getEnvironment()
                    .resolvePlaceholders(value);
        }
        return value;
    }
    @Override
    public void setResourceLoader(ResourceLoader resourceLoader) {
        this.resourceLoader = resourceLoader;
        // 注意，因为SpringMvcContract 也使用了 resourceLoader，所以必须给它指定解析器，否则不会解析占位符
        super.setResourceLoader(resourceLoader);
    }
}

@FeignClient(name = "feign",url = "127.0.0.1:8080")
// 使用的时候直接对 接口做 header的配置即可
@Headers({"app: test-app","token: ${test-app.token}"})
public interface FeignTest {
    @RequestMapping(value = "/test")
    String test();
}
```
### 优点
可以根据自己的需要自由定义

### 缺点
自定义带来一定的学习成本，而且因为是直接继承 spring 的实现，为以后升级留下隐患

## 方案四：在接口上使用 @RequestMapping，并加上 headers 属性
聪明的读者也许在方案二的结尾就能反应过来：springcloud 支持*@RequestMapping*注解的 header，而该注解完全可以用在类上面!  

```java
@FeignClient(name = "feign",url = "127.0.0.1:8080")
@RequestMapping(value = "/",headers = {"app=test-app","token=${test-app.token}"})
public interface FeignTest {
    @RequestMapping(value = "/test")
    String test();
}
``` 
### 优点
完全不用自定义，原生支持

### 缺点
基本没有。  
可能对于有些不习惯在类上使用 *@RequestMapping* 注解的同学来说，有点强迫症，不过基本可以忽略

### 思考：为什么没有一开始想到将注解放到接口定义那里
1. 思维定势，工作内容问题，很少会在feign接口上使用 *@RequestMapping*
2. SpringMvcContract 中的 *processAnnotationOnClass* 方法中没有关于对 header的处理，导致一开始忽略这个
3. SpringMvcContract 是在 *parseAndValidatateMetadata* 中解决类上面的 header 的问题

## 总结
1. 本文主要是探讨了 Contract 的一些功能，以及 springcloud 对它的一个处理
2. 网上很多在说 *@Headers* 无效，但是基本上都没说原因，这里对它做一个解释
3. 绕了一圈，还是回归到最简单的办法，使用 *@RequestMapping*