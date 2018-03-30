---
layout: post
title: 定长不重复随机字符串生成
categories: 日常工具
description: 定长不重复随机字符串生成
keywords: random, String
---

# 需求
做一个比赛的后台，参赛者报名成功后返回一个参赛码作为报名凭据，要求：
- 参赛码由大写字母和数字组合而成
- 参赛码必须 6 位
- 不同队伍的参赛码不同

# 分析
1. 需求没有明说，但是极端情况，想必 6 位的纯数字或者纯字母也是可以的
2. 6 位的数字和字母，可以有 36*36*36*36*36*36 = 2176782336 种组合，绝对够用
3. 参赛码不能相同，那就不能用单纯的随机了

# 方案
## 方案一
使用报名表的 ID，不足 6 位，左边补 0

### 优点
- 简单
- 不重复

### 缺点
- 没有字母
- 生成的参赛码是连续的，很容易被猜到
- 6 位数字，ID 跳几下就有可能冲上去了，如果 ID 有 7 位，就会重复

## 方案二
用随机生成参赛码，然后去数据库里查一下重复，如果重复就重新生成

### 优点
- 思路简单
- 36^6 大概 21 亿个组合，重复的概率还是很小的
- 参赛码作为凭据，肯定定义了主键

### 缺点
- 每重复一次，就要多查一次数据库
- 越往后重复的概率越大

## 方案三
如果要保证唯一，用 ID 是最好的方式，但是 ID 是数字类型，要怎么把数字转成字符呢？

**进制**

16 进制是 0～F，那么 36 进制就是 0～Z 了
```java
@Test
public void test3(){
    System.out.println(Integer.toString(1,36));  // 输出1
    System.out.println(Integer.toString(35,36)); // 输出z，记得转大写
    System.out.println(Integer.toString(36,36)); // 输出10
}
```

将 ID 转成 36 进制，刚好就是数字和字母的组合，然后不足 6 位的用 0 补齐

### 优点
- 简单
- 不重复

### 缺点
- 连续的，容易被猜到
- 一开始时，生成的数据前五位都是0
- 0 和 O,1 和 I 容易被用户混淆，建议排除

## 方案四
0～9A～Z中，去掉容易混淆的 0和O，1和I，刚好还剩下 32 个字符，可以按照 Integer.toString(int i, int radix) 的源码，实现一个自己的进制转换方法
```java
// 将正整数转换成 32 进制,注意数字不是从 0 开始的
private final static char[] digits = "23456789ABCDEFGHJKLMNPQRSTUVWXYZ".toCharArray();
private static String toDuoString(int i) {
    if(i < 0 ){
        throw new RuntimeException("请传入正整数");
    }
    char buf[] = new char[7];
    int charPos = 6;
    i = -i;
    while (i <= -32) {
        buf[charPos--] = digits[-(i % 32)];
        i = i / 32;
    }
    buf[charPos] = digits[-i];

    return new String(buf, charPos, (7 - charPos));
}

@Test
public void test4(){
    System.out.println(toDuoString(1));  // 输出3
    System.out.println(toDuoString(31)); // 输出Z
    System.out.println(toDuoString(32)); // 输出32
}
```
### 优点
- 把易混淆数据筛出，提高用户体验
- 数据不重复

### 缺点
- 连续的，拿到三个参赛码就能猜到
- 一开始生成的数据，前五位都是2，不够随机
- 如果需求确认必须字母和数字的组合，则不满足需求

## 方案五
1. 将 digits 中字符顺序打乱，降低被猜中概率
2. 进一步缩小参赛码范围至 32^4, 剩下的两位一位生成字母，一位生成数字
3. 将 ID 转换成 2 进制，然后转化成 20 位二进制字符串，不足的左边补 0
4. 将二进制字符串切成 4 段，每段 5 位，转成十进制后，去取对应的 digits 元素

```java
public class CodeMaker {
    private static final char[] baseArray = "Q2WE3R4T5YU6PA7S8DFG9HJKLZXCVBNM".toCharArray();
    private static final char[] numArray  = "23456789".toCharArray();
    private static final char[] letterArr = "ABCDEFGHJKLMNPQRSTUVWXYZ".toCharArray();
    private static final Random random = new Random();

    // 32*32*32*32 总共支持一百万的 id，超过就会重复
    public static String makeSignCode(Integer id){
        String idStr = Integer.toBinaryString(id);

        String binary = StringUtils.leftPad(idStr,20,'0');

        StringBuilder sb = new StringBuilder();
        sb.append(letterArr[random.nextInt(24)]);                        // 第一位是随机生成字母，保证字符串中有字母
        sb.append(baseArray[Integer.parseInt(binary.substring(0,5),2)]);  // 第二位是 ID 的 0-5位
        sb.append(baseArray[Integer.parseInt(binary.substring(5,10),2)]); // 第三位是 ID 的 5-10 位
        sb.append(baseArray[Integer.parseInt(binary.substring(10,15),2)]);// 第四位是 ID 的 10-15 位
        sb.append(baseArray[random.nextInt(32)]);                         // 第五位是随机生成数字，保证字符串中有数字
        sb.append(baseArray[Integer.parseInt(binary.substring(5,10),2)]); // 第六位是 ID 的 15-20 位

        return sb.toString();
    }
```
### 优点
- 不连续，接近随机
- 灵活，如果需求变化，直接修改每个字符位的组合即可

### 缺点
- 实现复杂，基本上都是自己实现
- 有效位数减小了两位, 不过对于本次需求来说，还是绰绰有余了

# 后记
1. 对于数据量小的情况，个人觉得方案二(随机生成，然后做重复校验)的方式也不错
2. 目前的方式，也还是需要多读一次数据库，去拿入库的 ID

## 扩展
又有个新需求，随机生成 4 位纯数字，不能跟历史数据重复
## 方案
考虑到 4 位数字不算多，可以使用穷举法：
1. 启动时将数据库中所有已经生成的数字拿出来 list1
2. 生成 1000-9999 的数字列表 list2
3. 将 list2 和 list1 共有的数据清除
4. 生成数据时，随机从 list2 中返回元素，然后将该元素 remove 掉。
