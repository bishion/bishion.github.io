---
layout: post
title: Activiti 6.0 用户文档 - 第十二章：Eclipse 设计器
categories: activiti
description: 流程引擎 Activiti 第十二章：Eclipse 设计器
keywords: 流程引擎, activiti, designer
---
# Eclipse 设计器
Activiti有一个Eclipse插件：Eclipse Activiti Designer，你可以使用它设计、测试和发布 BPMN2.0流程。
## 安装
下面的安装介绍在 [Eclipse Kepler和Indigo](http://www.eclipse.org/downloads)版本上已经测过了。注意，该插件 **不支持**Eclipse Helios。  
选择 **HELP** --> **Install New Software**，在弹出的面板中点击 *Add...* 按钮，然后填写如下值：
- Name：Activiti BPMN 2.0 designer
- Location：http://activiti.org/designer/update

![Activiti设计器site](/images/activiti/designer.add.update.site.png)  
确保 **Contact all updates sites..**选择框 **被勾选**，这样Eclipse才会下载所有需要的插件。
## Activiti 设计器功能
- 创建Activiti 项目和图表、


![Activiti设计器创建项目图](/images/activiti/designer.create.activiti.project.png)
- 当新建一个新的Activiti图表的时候，Activiti设计器会创建一个 .bpmn 文件。使用Activiti图表编辑器打开该文件，就可以看到图形建模画布和工具箱。该文件还可以使用XML编辑器打开，此时我们就可以看到流程定义的BPMN2.0 XML 元素。Activiti设计器支持对文件的图形化和BPMN2.0XML两种形式的操作。注意，Activiti5.9版本并不支持发布.bpmn后缀的流程定义。所以，Activiti设计器的 "create deployment artifacts"功能会生成一个带有.bpmn.xml文件的BAR文件，里面还包含一个.bpmn文件。你还可以给该文件重命名。另外还要注意，你也可以使用 Activiti 图形化编辑器打开一个 .bpmn20.xml 文件。  

![设计器打开bpmn图](/images/activiti/designer.bpmn.file.png)  
- Activiti 设计器中可以导入一个 BPMN2.0 XML 文件，然后会为它创建一个图形化界面。你只需要将BPMN2.0 XML文件放到你的项目中，然后选择使用Activiti图形化编辑器打开即可。如果你的BPMN2.0 XML文件里并没有DI信息，就不会创建图形化界面。

![设计器打开导入的bpmn图](/images/activiti/designer.open.importedfile.png)
- 如果想要部署一个BAR文件或者一个Activiti设计器创建的JAR文件，可以在 Eclipse的包浏览器下右键Activiti 项目，然后在弹出菜单的底部选择 *Create deployment artifacts*。有关设计器的部署功能的更多信息，请参阅[发布](https://www.activiti.org/userguide/#eclipseDesignerDeployment)章节。

![设计器创建一个发布图](/images/activiti/designer.create.deployment.png)
- 生成单测(在BPMN2.0XML文件上面右键，然后选择 *generate unit test*).设计器就会生成一个带有Activiti配置的单测用例，该用例是在H2数据库上执行的。然后你就可以运行该测试用例，来验证你的流程定义。  

![生成一个单测图](/images/activiti/designer.unittest.generate.png)
- Activiti 项目是一个Maven项目。你可以执行 *mvn eclipse:eclipse* 来配置它的依赖。注意，这里我流程设计并不需要Maven依赖，只有单测需要。

![项目maven结构图](/images/activiti/designer.project.maven.png)
## Activiti 设计器的BPMN功能
- 支持无事件启动、错误事件启动、定时事件启动、无事件结束、错误事件结束、顺序流、平行网关、排它网关、包容网关、事件网关、嵌入式子流程、事件子流程、调用活动、池、泳道、脚本任务，用户任务、服务任务、邮件任务、手动任务、业务规则任务、计时器边界事件、错误边界事件、信号边界事件、计时器捕获事件、信号捕获事件、信号抛出事件、未抛出事件、四个Alfresco特定元素（用户、脚本、邮件任务和开始事件）。  

![流程模型图](/images/activiti/designer.model.process.png)
- 可以快速修改任务类型：鼠标在元素上面悬停，并选择新的任务类型。  

![快速修改任务类型](/images/activiti/designer.model.quick.change.png)
- 可以快速添加新的元素：鼠标在元素上面悬停，并选择一个新的元素类型。

![快速新建元素](/images/activiti/designer.model.quick.new.png)
- 对于Java服务任务，支持Java类、表达式或者代理表达式配置。另外用户也可以配置属性表达式。

1[Java属性表达式](/images/activiti/designer.servicetask.property.png)
- 支持池和泳道。Activiti将不同的池看做不同的流程定义，所以用户最好只使用一个池。如果你使用多个池，请注意，Activiti 在发布流程的时候，池之间的顺序流将会导致问题。你可以在一个池中使用任意数量的的泳道。  

![泳道流程图](/images/activiti/designer.model.poolandlanes.png)
- 你可以填写顺序流的name属性，从而给它添加标签。你可以将该标签设置在任意地方，它会被当做BPMN2.0 XML DI信息的一部分被保存。  

![顺序流的标签](/images/activiti/designer.model.labels.png)
- 支持事件子流程

![事件子流程](/images/activiti/designer.model.eventsubprocess.png)
-支持嵌入式子流程。你还可以在一个嵌入式子流程中增加另一个嵌入式子流程。  
![嵌入式子流程](/images/activiti/designer.embeddedprocess.canvas.png)
- 支持给任务和嵌入式子流程添加计时器边界事件。当然了，在Activiti设计器中，在用户任务或嵌入式子任务上使用计时器边界事件最有意义。

![计时器边界事件](/images/activiti/designer.timerboundary.canvas.png)
- 支持 Activiti 扩展，比如邮件任务、用户任务赋权配置和脚本任务配置。

![邮件任务配置](/images/activiti/designer.mailtask.property.png)
- 支持 Activiti 执行和任务监听。你还可以给监听器执行方法添加属性扩展。

![监听器配置](/images/activiti/designer.listener.configuration.png)
- 支持顺序流条件

![顺序流条件](/images/activiti/designer.sequence.condition.png)
## Activiti 设计器的部署功能
