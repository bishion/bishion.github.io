---
layout: post
title: 使用 freeMarker 解析字符串
categories: bug, freeMarker
description: 使用 freeMarker 解析字符串，并发现里面有一个坑
keywords: freeMarker, StringTemplateLoader
---
# 背景
最近在做一个 java 代码脚手架，有用到 freeMarker 做模板引擎。脚手架中有一个功能，是需要根据传入的 groupId 来生成代码的包路径，该功能就需要用 freeMarker 解析文件的路径（字符串格式）。

# 实现
实现思路很简单，就是根据模板中的文件路径和一定的规则，使用freeMarker生成目标文件的路径。
我们先看下使用 freeMarker 解析字符串的代码实现：

```java
/**
 * freeMarker 版本：2.3.31
*/
public static void main(String[] args) {
    String text = "/tmp/${groupId}/Hello.java";
    Map<String, String> dataModel = new HashMap<>();
    dataModel.put("groupId","cn.bishion");

    Configuration cfg = new Configuration(Configuration.DEFAULT_INCOMPATIBLE_IMPROVEMENTS);
    StringTemplateLoader templateLoader = new StringTemplateLoader();
    templateLoader.putTemplate(text, text);                             // 1
    cfg.setTemplateLoader(templateLoader);                              // 2

    cfg.setDefaultEncoding("UTF-8");
    StringWriter writer = new StringWriter();
    try {
        cfg.getTemplate(text).process(dataModel, writer);               // 3
        System.out.println(writer);
    } catch (TemplateException | IOException e) {
        e.printStackTrace();
    }
}
```
# 遇到问题
如果你执行了上述代码，你会发现程序运行报如下错误：
```java
freemarker.template.TemplateNotFoundException: Template not found for name "/tmp/${groupId}/Hello.java".
The name was interpreted by this TemplateLoader: StringTemplateLoader(Map { "/tmp/${groupId}/Hello.java"=... }).
	at freemarker.template.Configuration.getTemplate(Configuration.java:2883)
	at freemarker.template.Configuration.getTemplate(Configuration.java:2685)
	at cn.bishion.scaffold.service.FreeMarkerTest.main(FreeMarkerTest.java:26)
```
# 错误解析
异常抛出点在上文源码段注释标注为 *3* 的位置。看异常内容是说，找不到 name 为 */tmp/xxx* 的模板。

但是，在源码段注释标注为 *1* 的地方，我们已经设置了变量 *text* 的值既作为模板名，又作为模板内容：
```java
templateLoader.putTemplate(text, text); 

// StringTemplateLoader.java 对应源码如下：

public void putTemplate(String name, String templateContent) {
    putTemplate(name, templateContent, System.currentTimeMillis());
}

```

目前，报错很诡异，但是根据错误内容，我们有两个排查方向：
1. put 的时候没有成功
2. get 的时候没有成功

翻看源码，我们发现：

**注释1代码**：将变量 *text* 的值作为 name 和模板，放入 *StringTemplateLoader* 的缓存中
**注释2代码**：将 *TemplateCache* 跟  *StringTemplateLoader* 建立关联
**注释3代码**：根据 *text* 的值，从 *TemplateCache* 中获取模板内容

重点在注释3中的代码
```java
// Configuration.java Line-2826
public Template getTemplate(String name,...){
        // 省略部分代码
        // 这里从缓存中获取模板，并做好缓存不存在的准备，源码见下文
        final MaybeMissingTemplate maybeTemp = cache.getTemplate(name, locale, customLookupCondition, encoding, parseAsFTL);
        final Template temp = maybeTemp.getTemplate();
        if (temp == null) {
            // 这里就报错。。。
        }

// TemplateCache.java Line-270
// 这里省略部分无关代码
public MaybeMissingTemplate getTemplate(String name,...){
    
        // 这里会对 name 做校验，并更新了 name 的值。
        // 看注释可知，该方法是对模板名称做一个标准化：将绝对路径变成相对路径，即，标准化之后的 name，第一位肯定不是 '/’
        name = templateNameFormat.normalizeRootBasedName(name);

        Template template = getTemplateInternal(name, locale, customLookupCondition, encoding, parseAsFTL);
        return template != null ? new MaybeMissingTemplate(template) : new MaybeMissingTemplate(name, (String) null);
    }    

```

# 问题确认
1. 在解析模板文件时，会将模板文件的相对 base 文件夹路径作为缓存 name，为处理方便，name 首位如果为 '/'，就去掉
2. 在解析模板字符串时，模板名必填，并作为 StringTemplateLoader 中的缓存的 name
3. freeMarker 使用统一的模板缓存工具 TemplateCache，仍会做上述处理，获取缓存时，将 name 标准化
3. 这就导致，存入缓存时的 name 跟获取缓存时的 name，已经不是一个值

# 解决方式
1. 尽量避免使用路径作为 StringTemplateLoader.putTemplate() 中的 name 参数
2. 如果做不到第1条，则 name 不要以 '/’ 开头
3. 鉴于当前脚手架项目的特殊性，程序选择去掉将 *templateNameFormat.normalizeRootBasedName(text)* 的执行结果做为 name 参数