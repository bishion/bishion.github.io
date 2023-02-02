---
layout: post
title: 使用 hutool MailUtil 发送邮件配置
categories: diary
description: 使用 hutool MailUtil 发送邮件配置, 飞书企业邮箱和腾讯企业邮箱不同
keywords: hutool, MailUtil, feishu, qq
---
# 使用 hutool MailUtil 发送邮件配置

## 官方文档参考代码

```java
MailAccount account = new MailAccount();
account.setHost("smtp.yeah.net");
account.setPort("25");
account.setAuth(true);
account.setFrom("hutool@yeah.net");
account.setUser("hutool");
account.setPass("q1w2e3"); // 密码（注意，某些邮箱需要为SMTP服务单独设置授权码，详情查看相关帮助）

MailUtil.send(account, CollUtil.newArrayList("hutool@foxmail.com"), "测试", "邮件来自Hutool测试", false);

```

## 腾讯企业邮箱

### 参考代码
```java

MailAccount account = new MailAccount();
account.setHost("smtp.qq.com");
account.setPort(465);
account.setAuth(true);
account.setStarttlsEnable(true); // 必须

account.setFrom("hutool@xxx.com"); // 发件人（必须正确，否则发送失败，报：Invalid Addresses: null）
account.setUser("hutool");         // 用户名，默认为发件人邮箱前缀，可以不填，也可以填发件人邮箱全称
account.setPass("q1w2e3");         // 腾讯邮箱授权码

MailUtil.send(account, CollUtil.newArrayList("hutool@foxmail.com"), "测试", "邮件来自Hutool测试", false);

```

### 常见报错
**Invalid Addresses: null**

from 字段非标准邮箱格式

**Connection or outbound has closed**
from 字段并非真实的发件人邮箱

**535 Login Fail. Please enter your authorization code to login**
user 填写错误，正确格式为：
1. 发件人邮箱前缀（缺省值）
2. 发件人邮箱

## 飞书企业邮箱

### 参考代码
```java

MailAccount account = new MailAccount();
account.setHost("smtp.feishu.cn");
account.setPort(465);
account.setAuth(true);
account.setStarttlsEnable(true); // 必须

account.setFrom("hutool@xxx.com"); // 发件人，可以随便填，作为发件人名称
account.setUser("hutool@xxx.com"); // 用户名, 必须为发件人邮箱全称
account.setPass("q1w2e3");         // 飞书邮箱授权码

MailUtil.send(account, CollUtil.newArrayList("hutool@foxmail.com"), "测试", "邮件来自Hutool测试", false);

```

### 常见报错
**AuthenticationFailedException: 535 Error: authentication failed, system busy**

user 字段未填写为发件人邮箱全称

## 总结
### 腾讯和飞书邮箱认证时，使用的字段不一样：
1. 腾讯邮箱使用 from + pass 来作为认证方式
2. 飞书邮箱使用 user + pass 来作为认证方式
### 飞书使用 from 字段作为发件人的名称
1. 如果 from 是邮箱格式，则废弃不用，发件人姓名为邮箱登记名称
2. 如果 from 是非邮箱格式，则直接作为发件人名称