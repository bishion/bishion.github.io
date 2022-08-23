---
layout: post
title: 运管页面导出报表的几种方式
categories: export, excel
description: 介绍在云管端导出报表的几种方式
keywords: operation, export, report, excel
---
# 运管页面导出报表功能设计

数据报表，应该是开发过程中最常见的功能需求了。对于一些大的公司，使用BI+数仓的方式，就能搞定几乎所有的导出场景；但是，对于一些小的公司，就要好好琢磨琢磨怎么办才好了。

## 太长不看版
1. 方案1: 最原始方式，每个导出一个接口
2. 方案2: 初步抽象，封装统一的导出接口
3. 方案3: 复用分页查询接口，增加一个注解来处理导出，并支持大量数据异步导出
4. 方案4: 独立一个导出平台，单独处理导出，并支持进度条
5. 方案5: 使用 mybatis 原理，导出平台解析动态 sql，与业务系统完全独立

## 方案一：面向CSDN编程
如果拿到需求，网上搜下“springmvc excel” 文件下载，估计代码一大堆，很多都能直接用，前后不超过一小时：
```java
    @PostMapping(value = "/export/order")
    public void orderExport(@RequestBody(required = false) QueryOrderReq queryOrderReq,
                                HttpServletResponse response) {
        try {
            List<ExportOrderDTO> exportList = orderService.queryOrderExportReq(queryOrderReq);
            try {
                response.setContentType("application/vnd.ms-excel");
                response.setCharacterEncoding("utf-8");
                response.setHeader("Content-disposition", "attachment;filename=订单明细.xlsx");
                EasyExcel.write(response.getOutputStream(), ExportOrderDTO.class).sheet("明细").doWrite(exportList);
            } catch (Exception e) {
                log.info("订单导出 失败{}", e.getMessage(), e);
                throw new BizException(CommonError.SYS_ERROR, "订单导出失败!");
            }
        } 
    }   
```

### 评价
1. 一个报表写个方法，简单快捷，不用动脑子
2. emmmm，这就是传说中的将代码行数作为KPI的团队？

## 方案二：功能抽象
导出功能的核心需求就是：
1. 根据查询条件查询数据
2. 将数据生成 excel 文件
3. 写入响应流，给用户下载
所以，我们很容易想到将生成文件以及下载功能独立。于是就有了下面的代码(因脱敏可能有编译运行错误)：

```java
public enum ExportTypeEnum {
    NORMAL_ORDER("normalOrderExportService",NormalOrderExportDTO.class,"订单明细.xlsx"),
    ORDER_WITH_PRODUCT("orderWithProductExportService",NormalOrderExportDTO.class,"订单明细(带商品).xlsx");
}

public interface ExportService {

    // 根据条件查询出数据总条数
    Long count(Map cond);
    // 根据条件和起始记录，查询一页条数
    List selectByPage(Map cond, Integer start, Integer pageSize);

}
@RestController
public class DataExportController {

    @Resource
    private Map<String, ExportService> exportServiceMap;

    @PostMapping("/export/${exportType}")
    public void export(@RequestBody Map<String,Object> param, @PathVariable("exportType") String exportType, HttpServletResponse response){
        ExportTypeEnum exportTypeEnum = EnumUtil.fromString(ExportTypeEnum.class, exportType);
        ExportService exportService = exportServiceMap.get(exportTypeEnum.getServiceName());

        Long count = exportService.count(param);

        try (ServletOutputStream out = response.getOutputStream()) {
            //设置字符集为utf-8
            response.setCharacterEncoding("utf8");
            response.setContentType("application/vnd.ms-excel;charset=utf-8");
            //通知浏览器服务器发送的数据格式
            response.setHeader("Content-Disposition", "attachment; filename=" + URLEncoder.encode(exportTypeEnum.getFileName(), "UTF-8"));

            ExcelWriter excelWriter = EasyExcel.write(out, exportTypeEnum.getClazz()).build();
            // 这里注意 如果同一个sheet只要创建一次
            WriteSheet writeSheet = EasyExcel.writerSheet("sheet").build();
            if (count > 0){
                for (int i = 0;i < count; i=i+500){
                    List records = exportService.selectByPage(param,i,500);
                    excelWriter.write(records, writeSheet);
                    records.clear();
                }
            }
            excelWriter.finish();
            out.flush();
        } catch (IOException e) {
            throw new RuntimeException();
        }
    }

}

```
### 优点
1. 将导出功能收为一个入口，消除重复劳动
2. 新加报表的话，只需要扩展枚举以及添加对应的查询实现类（其实，枚举也可以拿掉）
### 缺点
1. 在数据量比较大的时候，请求会超时
2. 不同的报表类型，每页查询的数据可能会有不同
3. 绝大多数的导出，本身都对应着一个分页查询的接口，可否直接复用
4. 既然已经抽象如斯，是否可以更进一步，做成工具包


## 方案三：独立的导出组件
根据方案二的缺点，组内小朋友提出进一步优化方案：
1. 写一个注解，直接挂在分页查询的 web 接口上，并开启切面
2. 注解中指定各种参数，比如文件名，每页条数，最大实时导出条数等
3. 当查询参数中指定 exportFlag=true 时，则执行导出逻辑
4. 如果查询结果超过指定值，则异步导出报表并发送到用户邮箱
5. 封装成工具包，客户端按需引用

示例代码如下：
```java
public @interface ExcelExport {
    // 导出到Excel的实体
    Class targetType();

    // 每页大小
    int pageSize() default 5000;

    // 同步下载最大条数
    int syncMaxSize() default 50000;

    // 文件名
    String title();

}

 @Around("exportPointCut()")
    public Object process(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        ExportBaseCond exportBaseCond = null;
        for (Object obj : proceedingJoinPoint.getArgs()){
            if (obj instanceof ExportBaseCond){
                exportBaseCond = ((ExportBaseCond) obj);
                break;
            }
        }
        Method method = ((MethodSignature) proceedingJoinPoint.getSignature()).getMethod();
        // 如果参数不符合要求，则直接返回
        if (Objects.isNull(exportBaseCond) || BooleanUtil.isFalse(exportBaseCond.getExportFlag()) ||
        !(method.getReturnType() == PageResult.class)){
            return proceedingJoinPoint.proceed();
        }

        Object result = proceedingJoinPoint.proceed();
        PageResult pageResult = ((PageResult<?>) result);
        ExcelExport export = method.getAnnotation(ExcelExport.class);
        // 注意：这里只考虑了实时导出的代码，异步的导出可以自行扩展
        OutputStream out = buildOutStream(export);

        ExcelWriter excelWriter = EasyExcel.write(out, export.targetType()).build();
        // 这里注意 如果同一个sheet只要创建一次
        WriteSheet writeSheet = EasyExcel.writerSheet("sheet").build();
        Long count = pageResult.getTotal();
        if ( count> 0){
            long pages = count/export.pageSize()+1;
            exportBaseCond.setPageSize(export.pageSize());
            for (int i = 0;i < pages; i++){
                exportBaseCond.setCurrentPage(i);
                pageResult = (PageResult) proceedingJoinPoint.proceed();
                excelWriter.write(pageResult.getRecords(), writeSheet);
            }
        }
        // 千万别忘记finish 会帮忙关闭流
        excelWriter.finish();
        out.flush();
        return null;
    }
```

### 优点
1. 做成工具包，可以实现开箱即用
2. 支持自定义实时或者异步导入导出
3. 复用现有的分页查询接口, 无需额外编写代码
### 缺点
1. 分页查询接口每次调用都会执行 count，而我们只需要一次
2. 关于异步导出，强依赖了发邮件功能（或者其他三方功能，总之需要额外配置）
3. 二方库的升级，在业务开发团队很难推行
4. 无法查看导出进度

## 方案四：独立的导出平台
既然可以使用注解的方式复用分页查询接口，那我完全可以将切面中的逻辑直接独立到一个单独的应用中。

好吧，其实是前司的方案，这里致敬一下：
1. 将导出做成一个完全独立的应用，并作为异步功能
2. 前端将导出请求发给平台，平台返回一个 uuid
3. 平台异步请求应用的接口，分页捞取数据，并将捞取进度存到缓存，key为 uuid
4. 前端使用该 uuid 进行进度轮询
5. 等到捞取完毕后，平台返回导出的数据（可以是文件流，也可以是文件链接）

### 优点
1. 文件导出完全独立，可以针对性优化升级
2. 用户可以直接看到导出进度

### 缺点
1. 在很多场景中，导出的数据总会比查到页面上的多，甚至会关联更多的表，因此不建议复用
2. 碰到突然加个需求，多导一个字段，还是要发布上线

## 方案五：更加独立的导出平台
气氛都烘托到这儿了，那就摊牌了：直接将导出的sql 也放到导出平台吧。什么注解，什么接口复用都不要了：
1. 将导出功能完全独立，与业务系统无关
2. 导出平台配置模板sql，动态解析前端参数并生成sql查询语句
3. 技术方案参考 mybatis 的 mapper.xml 解析过程

### 优点
1. 与日常分页查询接口独立，可以各自优化性能
2. 增减导出字段只需要修改模板，无需发布

### 缺点
1. 查询条件变化的话需要跟着一起调整sql模板，不然无法做到所见即所导
2. 平台需要管理多个数据源（也可以接受）
3. 要是查询条件也能支持页面拖拽那就更好了（这不就是 BI 么）
4. 该平台就是一个低配版BI，如果公司买了 BI 相关产品，它也就没用了