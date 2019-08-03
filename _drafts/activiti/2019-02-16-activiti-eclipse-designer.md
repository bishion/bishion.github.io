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
在Activiti 引擎上发布流程定义和任务表单并不难，你只需要一个包含流程定义的BPMN2.0 XML文件或许还有任务表单和流程图片的BAR文件。你可以使用Activiti 设计器很容易第创建BAR文件。当你完成流程定义之后，右键点击Activiti项目，然后在菜单底部选择 **Create deployment artifacts**即可。  
![创建发布组件](/images/activiti/designer.create.deployment.png)  
此时，在你的Activiti项目中就会出现一个 *deployment* 文件件，里面有一个BAR文件，或许还会有一个 JAR包。  
![发布包目录](/images/activiti/designer.deployment.dir.png)  
你可以在Activiti Explorer的发布一栏将该文件上传至Activiti引擎。  
如果你的项目包括Java文件，Activiti设计在**Create deployment artifacts**这一步还会生成一个JAR文件，里面包含编译过的类。该JAR包需要被放到你Tomcat 下的Activiti-XXX/WEB-INF/lib目录下，这样你的Activiti引擎才能在类路径下访问到这些类。
## 扩展Activiti 设计器
你可以对Activiti设计器的默认功能进行扩展。本节将介绍你可以做哪些扩展，它们怎么使用以及我们做的一些示例。当给业务流程进行建模的时候，如果Activiti设计器默认的功能不符合你的要求，你需要更多的特性或者在特定领域有需求，那么对设计器的扩展就很有用了。对设计器的扩展最终会落到两个完全不同的方向：扩展工具面板和扩展输出格式。这两个方向的扩展需要不同的途径和不同的技术栈。
> 扩展Activiti设计器需要专业的知识和更专业的Java编程水平。你需要熟悉Maven，Eclipse，OSGI，Eclipse 扩展和SWT中的部分或全部，具体取决于你要扩展哪部分的功能。

### 自定义工具面板
你可以自定义流程建模时显示的那个工具面板。该工具面板位于画布的右手边，是一组可以拖拽到流程画板的形状。对于默认的工具面板，你可以看到形状被按组放到不同的栏目里（我们称为"抽屉“），有事件组、网关组等等。Activiti设计器内置两个选项来自定义工具面板的抽屉和形状：
- 将你自己的形状/节点添加到现有或新增的抽屉
- 禁用Activiti设计器中部分或者所有BPMN2.0默认的形状，除了连线和选择工具。

要想自定义工具面板，你需要创建一个JAR包，然后把它加到Activiti设计器的安装路径中（更多的可以参考[如何操作](https://www.activiti.org/userguide/#eclipseDesignerApplyingExtension)）。该JAR包就是前文所说的 *扩展*。通过那些扩展的类，Activiti设计器就知道你做了哪些自定义。你需要实现特定的接口来让它生效。我们提供了一个集成库，里面包含接口和基础类，你可以将它引用到你的项目中。  
你可以使用Activiti设计器在源码管理中看到下面的代码实例。打开Activiti的源码，在 **projects/designer** 目录下的 **examples/money-tasks**就可以找到。
> 你可以自由选择编写扩展项目的IDE。接下来的介绍，我们使用的是Eclipse Kepler或者Indigo、Maven(3.x)。

#### 扩展设置(Eclipse/Maven)
下载并解压[Eclipse](http://www.eclipse.org/downloads)(最近版本的一般都可以)和最近的[Apache Maven](http://maven.apache.org/download.html)(3.x)。如果你使用2.x版本的maven，在构建项目的时候，会报错，所以尽量确保你的版本是新的。我们假设你能熟练使用Eclipse的基本特定和Java编辑器。是选择Eclipse的Maven插件还是在命令行中执行Maven命令，随便你。

新建一个新的Eclipse 项目，可以是一个普通类型的项目。在项目根目录下新建一个 **pom.xml**，这是Maven项目的基本配置。然后创建 **src/main/java**和 **src/main/resources**目录，这些都是Maven对Java代码和资源的约定目录。打开 **pom.xml**，添加如下内容：
```xml
<project
  xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">

  <modelVersion>4.0.0</modelVersion>

  <groupId>org.acme</groupId>
  <artifactId>money-tasks</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>
  <name>Acme Corporation Money Tasks</name>
...
</project>
```
你能看到，这只是一个定义了 **groupId**，**artifactId** 和 **version** 的一个基础pom文件。我们会为我们的money 业务创建一个包含单个用户节点的自定义。

给 **pom.xml**文件添加集成库：
```xml
<dependencies>
  <dependency>
    <groupId>org.activiti.designer</groupId>
    <artifactId>org.activiti.designer.integration</artifactId>
    <version>5.12.0</version> <!-- Use the current Activiti Designer version -->
    <scope>compile</scope>
  </dependency>
</dependencies>
...
<repositories>
  <repository>
      <id>Activiti</id>
      <url>https://maven.alfresco.com/nexus/content/groups/public/</url>
   </repository>
</repositories>
```
最后，再给该pom添加 **maven-compiler-plugin**配置，这样Java源码版本就是1.5以上了(参考下面的代码片段)，你需要该设置来支持注解。你还可以给该Maven项目生成 **MANIFEST.MF**。这不是必须的，但是你可以在manifest设置一个特定的属性，从而给你的扩展提供一个名字(该名字会在设计器的特定位置显示出来，如果你定义了很多扩展的话，该功能就很有用了)。如果你想使用上述功能，就在你的  **pom.xml**文件中添加如下代码片段：
```xml
<build>
  <plugins>
        <plugin>
      <artifactId>maven-compiler-plugin</artifactId>
      <configuration>
        <source>1.5</source>
        <target>1.5</target>
        <showDeprecation>true</showDeprecation>
        <showWarnings>true</showWarnings>
        <optimize>true</optimize>
      </configuration>
    </plugin>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-jar-plugin</artifactId>
      <version>2.3.1</version>
      <configuration>
        <archive>
          <index>true</index>
          <manifest>
            <addClasspath>false</addClasspath>
            <addDefaultImplementationEntries>true</addDefaultImplementationEntries>
          </manifest>
          <manifestEntries>
            <ActivitiDesigner-Extension-Name>Acme Money</ActivitiDesigner-Extension-Name>
          </manifestEntries>
        </archive>
      </configuration>
    </plugin>
  </plugins>
</build>
```
扩展的名称通过 **ActivitiDesigner-Extension-Name**属性定义。接下来就只剩下一件事了：告诉Eclipse根据 **pom.xml**来设置当前项目。打开命令行工具，进入到项目的根目录，然后执行下面的 Maven 命令：
```shell
mvn eclipse:eclipse
```
等到构建成功，刷新项目(右键该项目，选择 **Refresh**)。你可以看到项目的 **src/main/java**和 **src/main/resources**目录都编程了资源目录。
> 你当然可以使用[m2eclipse](http://www.eclipse.org/m2e)插件来做这件事。右键该项目，选择 **Maven** > **Update project configuration**即可。

这样就全部设置完了，接下来你就可以自定义Activiti设计器功能了！
#### 给Activiti设计器应用你的扩展
你可能会好奇，你的扩展是怎么在Activiti设计器中添加并生效的。其实是分了这几步：
- 一旦生成了扩展JAR包(比如执行mvn install)，你需要将该jar包放到设计器的安装目录中
- 将该包保存在硬盘上并记住它的位置。注意：该目录不能与Eclipse的Activiti 设计器所在目录相同
- 启动Activiti设计器，选择 **Window** > **Preferences**
- 在preferences界面，输入 **user** 关键字，然后在 **Java**一栏选择 **User Libraries**

![用户库](/images/activiti/designer.preferences.userlibraries.png)
- 在右侧的User Libraries面板中，你能看到默认的分组(取决于Eclipse的安装方式，可能你会看到一些的分组)。接着，你就可以给Activiti设计器添加扩展了。

![用户库中空activiti包](/images/activiti/designer.preferences.userlibraries.activiti.empty.png)
- 选择 **Activiti Designer Extensions**，然后点击 **Add JARs…​** 按钮。在弹出框里找到你的扩展的存放路径，然后选中它。完成这些，你就能看到 **Activiti Designer Extensions**分组下面有你的扩展包，就像下面这样：
![用户库中包含你的扩展库](/images/activiti/designer.preferences.userlibraries.activiti.moneytasks.png)
- 点击 **OK**来保存并关闭对话框。**Activiti Designer Extensions**就会自动加入到你新创建的Activiti项目中了。你可以在Eclipse的Navigator或者Package视图中的看到用户库。如果你在工作空间里已经有Activiti项目了，在Activiti组里也能看到这个新扩展包。示例如下：
- 
![项目中包含扩展库](/images/activiti/designer.userlibraries.project.png)

现在你打开的流程图里就能看到你在工具面板扩展的形状了(或者是禁用的形状，这个取决于你做了哪些扩展)。如果已经有一个打开的流程图，你需要关闭重新打开就可以了。
#### 给工具面板添加新形状
等你的项目都设置好，就就可以很容易给工具面板添加新形状。每一个你想添加的形状，都对应着JAR包中的一个类。注意，这些类并不是流程引擎运行的时候使用到的类。Activiti 设计器可以根据你设定的属性来展示每一个形状。在这些形状中，你还可以定义一些运行时特性，这些特性将在流程定义到达某个节点时被用到。运行时特性可以使用Activiti支持普通 **ServiceTask** 的任何特性。点击[这里](https://www.activiti.org/userguide/#eclipseDesignerConfiguringRuntime)，查看更多详细信息.

形状的类就是一个普通Java类，只是多了一些注解。这个类需要实现 **CustomServiceTask**接口，但是你不需要自己来实现该接口，只需要继承 **AbstractCustomServiceTask**就好(你**必须**直接继承该类，中间不能再有其他抽象类)。在这个类的javadoc中，你可以看到对它的默认方法的说明，你可以重写它已经实现的任意方法。通过重写，你可以给工具面板提供图标和它在画板上的形状(这两个图可以不相同)以及指定您希望节点具有的基本形状(活动，事件，网关)。
```java
/**
 * @author John Doe
 * @version 1
 * @since 1.0.0
 */
public class AcmeMoneyTask extends AbstractCustomServiceTask {
...
}
```
你需要实现 **getName()**方法来为你的节点在工具面板中命名。你还可以将这个节点放在独立的抽屉中，并提供一个自定义图表，只需要重写 **AbstractCustomServiceTask**合适的方法。如果你想展示一个图标，确保它放在你JAR包的 **src/main/resources**目录下，大小为16*16像素，格式为JPEG或PNG格式。你提供的path是相对于刚刚的资源目录的相对路径。

你可以添加成员变量并使用 **@Property**注解来给形状添加属性。如下所示：
```java
@Property(type = PropertyType.TEXT, displayName = "Account Number")
@Help(displayHelpShort = "Provide an account number", displayHelpLong = HELP_ACCOUNT_NUMBER_LONG)
private String accountNumber;
```
有很多 **PropertyType**值供你使用，点击[这里](https://www.activiti.org/userguide/#eclipseDesignerPropertyTypes)查看它们的详细介绍。你可以通过将required属性设置为true，来让该值必填。如果用户不填写该值，就会弹出一条信息，输入框背景也会变红。

如果你想让属性按照类中的变量的顺序展示，那么就需要给该变量的 **@Property**注解指定order属性。

我们提供了 **@Help**注解，可以在用户填写值的时候展示提示信息。你还可以在类上面使用 **@Help**注解，这样提示信息就会放在属性表单的顶栏。

下面就是 **MoneyTask**的一个深入阐述，我们给该节点添加了一个comment属性和一个图标。
```java
/**
 * @author John Doe
 * @version 1
 * @since 1.0.0
 */
@Runtime(javaDelegateClass = "org.acme.runtime.AcmeMoneyJavaDelegation")
@Help(displayHelpShort = "Creates a new account", displayHelpLong = "Creates a new account using the account number specified")
public class AcmeMoneyTask extends AbstractCustomServiceTask {

  private static final String HELP_ACCOUNT_NUMBER_LONG = "Provide a number that is suitable as an account number.";

  @Property(type = PropertyType.TEXT, displayName = "Account Number", required = true)
  @Help(displayHelpShort = "Provide an account number", displayHelpLong = HELP_ACCOUNT_NUMBER_LONG)
  private String accountNumber;

  @Property(type = PropertyType.MULTILINE_TEXT, displayName = "Comments")
  @Help(displayHelpShort = "Provide comments", displayHelpLong = "You can add comments to the node to provide a brief description.")
  private String comments;

  /*
   * (non-Javadoc)
   *
   * @see org.activiti.designer.integration.servicetask.AbstractCustomServiceTask #contributeToPaletteDrawer()
   */
  @Override
  public String contributeToPaletteDrawer() {
    return "Acme Corporation";
  }

  @Override
  public String getName() {
    return "Money node";
  }

  /*
   * (non-Javadoc)
   *
   * @see org.activiti.designer.integration.servicetask.AbstractCustomServiceTask #getSmallIconPath()
   */
  @Override
  public String getSmallIconPath() {
    return "icons/coins.png";
  }
}
```
如果你使用这个形状扩展Activiti设计器，工具面板和对应的节点就是如下这个样子的：  
![MoneyTaskNode展示](/images/activiti/designer.palette.add.money.png)

money task的属性界面如下所示。注意 **accountNumber**属性的必填提示：  
![必填属性提示](/images/activiti/designer.palette.add.money.properties.required.png)

在创建流程图的时候，用户可以在这些属性值上面填静态文本或者流程变量表达式(比如：这个小猪猪走进了${piggyLocation})。一般来说，用户可以在这些文本域中填写任何信息。如果你期望用户使用表达式，然后你在 **CustomServiceTask**(使用**@Runtime**)中使用运行时行为，请确保在委托类中使用 **Expression**属性，这样表达式才能在运行时被正确解析。更多的运行时行为信息请参考[这里](https://www.activiti.org/userguide/#eclipseDesignerConfiguringRuntime)。

该属性的提示信息会在点击右侧的按钮时显示，示例如下：  
![属性的help信息](/images/activiti/designer.palette.add.money.help.png)

#### 给自定义服务任务配置运行时执行
当属性设置完，设计器中也加入了扩展之后，用户可以在定义流程的时候配置这些服务任务的属性。大多数的情况下，你期望Activiti在执行流程的时候能用到这些用户配置的属性。想要达到这个效果，你必须告诉Activiti在流程到达你的 **CustomServiceTask**时候使用哪个类的实例。

我们提供了 **@Runtime**注解，你可以用它给你的 **CustomServiceTask**指定运行时特性。下面是它的一个例子：
```java
@Runtime(javaDelegateClass = "org.acme.runtime.AcmeMoneyJavaDelegation")
```
在流程建模过程中输出的BPMN中，你的 **CustomServiceTask**就变成了一个普通的 **ServiceTask**。Activiti支持[几种方式](https://www.activiti.org/userguide/#bpmnJavaServiceTask)来给 **TaskService**定义运行时特性。因此，**@Runtime**注解使用下面三种属性之一来跟Activiti一一对应，就像下面这样：
- **javaDelegateClass**映射为BPMN中 **javaDelegateClass**。需要指定一个 **JavaDelegate**实现类的全路径
- **expression**映射为 BPMN中**activiti:expression**。指定一个要执行的方法的表达式，比如一个 Spring Bean中的方法。使用该方法的时候 **一定不要**指定 **@Property**注解。
- javaDelegateExpression映射为BPMN中 **activiti:delegateExpression**。指定一个 **JavaDelegate**实现类的表达式。

如果你给Activiti注入带有成员变量的类，那么用户的属性就会被注入到运行时类中。这些属性名称需要跟 **CustomeServiceTask**中的变量名相同。更多的信息，可以点击[这里](https://www.activiti.org/userguide/#serviceTaskFieldInjection)。注意，设计器的5.11.0版本以后，你可以对动态属性值使用 **Expression** 接口。这表示，Activiti设计器中这个属性值必须含有一个表达式，而且这个表达式会被注入到**JavaDelegate**实现类的**Expression**属性上面。
> 你可以在你的 **CustomServiceTask**类变量上面使用 **@Property**注解，但是如果你使用 **@Runtime**的 **expression**属性就不行。原因在于你指定的表达式是要被Activiti解析为一个方法，而不是类。如果你使用 **@Runtime**中的 **expression**，所有 **@Property**注解的成员都会被设计器忽略。设计器不会将它们渲染成节点属性窗口的可编辑域，也不会输出为BPMN流程中的属性。

> 注意，运行时的类不能放在你的扩展 JAR 中，因为它依赖Activiti 库。Activiti 需要在运行时能访问它，所以这些类要放在流程引擎的类路径下。

设计器源码中的示例包括了对 **@Runtime**不同配置的例子。多看看 money-task项目，找找切入点。money-delegates项目中包含一些使用委托类的例子。
####属性类型
本节介绍了你可以在 **CustomServiceTask**的属性上设置的 **PropertyType**值。
##### PropertyType.TEXT
创建一个下图所示的单行文本域。可以设置为一个必填属性，并在悬浮框里展示校验信息。校验失败的话，就会将文本域的背景设置成淡红色。  
![文本校验](/images/activiti/designer.property.text.invalid.png)
##### PropertyType.MULTILINE_TEXT
创建一个下图所示的多行文本域(高度被限定在80像素)。可以设置为一个必填属性，并在悬浮框里展示校验信息。校验失败的化，就会将文本域的背景设置成淡红色。  
![文本域校验](/images/activiti/designer.property.multiline.text.invalid.png)。  
##### PropertyType.PERIOD
创建一个结构化的编辑器，通过使用微调控去编辑每一个单元的数量，从而指定一段时间。展示结果如下图。可以设置为必填字段(可以理解为至少有一个单元内的值为非0)，然后在悬浮框里展示校验信息。校验失败后，整个属性域背景都变成淡红色。属性的值会被设置成一个固定格式的字符串：1y 2mo 3w 4d 5h 6m 7s，这表示1年2月3周4天5小时6分钟7秒。整个字符串都会被保存，即使所有的单元都是0.  
![一段时间](/images/activiti/designer.property.period.png)
##### PropertyType.BOOLEAN_CHOICE
为boolean数据或者切换选项创建单个复选框。注意，你可以给该字段设置 **required**属性，但是系统不会真正去校验，因为这将使用户无法选择是否选中该框。该值在流程图中以 java.lang.Boolean.toString(boolean)保存，即 "true"或者"false"。  
![布尔选择框](/images/activiti/designer.property.boolean.choice.png)
##### PropertyType.RADIO_CHOICE
创建下图所示的一组单选框。选择任意一个单选按钮就会取消其他的选择(即：只允许选择一个)。可以设置成必填，并在悬浮框里显示校验信息。如果校验失败，该属性背景将变成淡红色。  
该属性要求被注解的成员变量还要加上一个 **@PropertyItems**注解(如下所示)。通过该注解，你可以指定一个数组来存放候选值。我们需要给这些项目添加两个数组的元素，一个是要显示的标签，第二个是要保存的值。
```java
@Property(type = PropertyType.RADIO_CHOICE, displayName = "Withdrawl limit", required = true)
@Help(displayHelpShort = "The maximum daily withdrawl amount ", displayHelpLong = "Choose the maximum daily amount that can be withdrawn from the account.")
@PropertyItems({ LIMIT_LOW_LABEL, LIMIT_LOW_VALUE, LIMIT_MEDIUM_LABEL, LIMIT_MEDIUM_VALUE, LIMIT_HIGH_LABEL, LIMIT_HIGH_VALUE })
private String withdrawlLimit;
```
![单选框](/images/activiti/designer.property.radio.choice.png)
##### PropertyType.COMBOBOX_CHOICE
创建一个下图所示的固定值下拉框。可以设置成必填，并在悬浮框里显示校验信息。如果校验失败，该下拉框背景将变成淡红色。  
该属性要求被注解的成员变量还要加上一个 **@PropertyItems**注解(如下所示)。通过该注解，你可以指定一个数组来存放候选值。我们需要给这些项目添加两个数组的元素，一个是要显示的标签，第二个是要保存的值。
```java
@Property(type = PropertyType.COMBOBOX_CHOICE, displayName = "Account type", required = true)
@Help(displayHelpShort = "The type of account", displayHelpLong = "Choose a type of account from the list of options")
@PropertyItems({ ACCOUNT_TYPE_SAVINGS_LABEL, ACCOUNT_TYPE_SAVINGS_VALUE, ACCOUNT_TYPE_JUNIOR_LABEL, ACCOUNT_TYPE_JUNIOR_VALUE, ACCOUNT_TYPE_JOINT_LABEL,
  ACCOUNT_TYPE_JOINT_VALUE, ACCOUNT_TYPE_TRANSACTIONAL_LABEL, ACCOUNT_TYPE_TRANSACTIONAL_VALUE, ACCOUNT_TYPE_STUDENT_LABEL, ACCOUNT_TYPE_STUDENT_VALUE,
  ACCOUNT_TYPE_SENIOR_LABEL, ACCOUNT_TYPE_SENIOR_VALUE })
private String accountType;
```
![下拉框](/images/activiti/designer.property.combobox.choice.png)
##### PropertyType.DATE_PICKER
创建一个下图所示的日期选择器。可以设置成必填项，并在悬浮框里显示校验信息（注意，该选择器会自动设置成系统当前时间，所以基本上不会为空）。校验失败的话，该选择器的背景将会变成淡红色。  
该属性要求被注解的成员变量还要添加一个 **@DatePickerProperty**注解(如下所示)。通过该注解，你可以指定日期的保存格式和日期选择器的展示类型。这两个属性为可选的，如果你不指定的话，系统会使用默认值(它们在 **DatePickerProperty**注解中是静态值)。**dateTimePattern**属性需要设置成一个 **SimpleDateFormat**类支持的类型。当使用 **swtStyle**属性，你需要指定一个 **SWT**中的 **DateTime**选择器支持的整型，因为这个属性就是使用该选择器渲染的。
```java
@Property(type = PropertyType.DATE_PICKER, displayName = "Expiry date", required = true)
@Help(displayHelpShort = "The date the account expires ", displayHelpLong = "Choose the date when the account will expire if no extended before the date.")
@DatePickerProperty(dateTimePattern = "MM-dd-yyyy", swtStyle = 32)
private String expiryDate;
```
![日期选择器](/images/activiti/designer.property.date.picker.png)
##### PropertyType.DATA_GRID
创建一个下图所示的表格控件。用户可以使用数据表格输入任意行的数据，而且每行数据的列是固定的(行和列交叉的部分称作单元格)。用户可以根据需要添加或删除一行。  
该属性还要跟 **@DataGridProperty**注解搭配使用(如下所示)。通过这个新注解，你可以指定数据表格的属性。你需要通过 **itemClass** 属性指定另外一个类，来对应数据表格的每一列。Activiti设计器将该成员类型定义为 **List**。按照管理，你可以使用 **itemClass** 属性指定的类作为它的通用类型。比如，你可以将你编辑的表格当做一个 groceryList，**GroceryListItem**类的属性对应着表格的每一列。在 **CustomServiceTask**中，你可以像这样表示它：
```java
@Property(type = PropertyType.DATA_GRID, displayName = "Grocery List")
@DataGridProperty(itemClass = GroceryListItem.class)
private List<GroceryListItem> groceryList;
```
"itemClass" 类跟你用来指定 **CustomServiceTask** 字段时使用的是相同的注解，但是使用数据表格时除外。特别地，我们支持**TEXT**，**MULTILINE_TEXT**，**PERIOD**。你已经注意到，表格中每一行都是一组文本控件，无论它们的 **PropertyType**是什么。这样处理的原因是保证表格的美观和可读性。举个例子，如果要按照常规格式展示 **PERIOD PropertyType**，你可以想象，不打乱界面的情况下，根本无法将它正确放入表格单元中。对于 **MULTILINE_TEXT**和 **PERIOD**这样的 **PropertyType**，双击对应的单元格，系统会为弹出一个更大的编辑框。当用户点击 OK，它的值就会保存到文本中，这样就保证了表格的可读性。

非空属性的处理跟 **TEXT**差不多，当任何一个单元格失去焦点的时候，系统就会对整个表格执行格式校验。如果某个单元格校验失败，它的背景就会变成淡粉色。

默认情况下，该组件允许用户新增行，但是并不允许它们修改行的顺序。如果你想放开该功能，需要将 **orderable**设置为true，这样在每一行的后面就会有一个按钮，用户通过点击它就可以将该行上下移动。
> 目前，该属性类型未能正确注入到运行时的类中。


![表格控件](/images/activiti/designer.property.datagrid.png)
#### 禁用工具面板中的默认形状
使用该功能你需要在你的扩展中添加一个类，并实现 **DefaultPaletteCustomizer**接口。你不应该实现该接口，而是继承 **AbstractDefaultPaletteCustomizer**。目前，该类不提供任何功能，但是未来版本中，**DefaultPaletteCustomizer**接口会提供更多特性，而这个抽象类也会默认实现它们，因此，你通过继承该类，你的扩展将能保证对未来版本的兼容性。

扩展 **AbstractDefaultPaletteCustomizer**需要你实现一个方法，**disablePaletteEntries()**，并返回个 **PaletteEntry**的列表。对于每一个你想禁用的默认形状，你就将相应的 **PaletteEntry**添加到返回的列表中。注意，如果你将某个抽屉的所有形状都移除了，那么该抽屉也会从面板中移除。如果你想将所有形状都移除，只需要将 **PaletteEntry.ALL**加入到你的返回值。下面这个代码就演示了如果在面板中禁用 *Manual task* 和 *Script task*两个形状。
```java
public class MyPaletteCustomizer extends AbstractDefaultPaletteCustomizer {

  /*
   * (non-Javadoc)
   *
   * @see org.activiti.designer.integration.palette.DefaultPaletteCustomizer#disablePaletteEntries()
   */
  @Override
  public List<PaletteEntry> disablePaletteEntries() {
    List<PaletteEntry> result = new ArrayList<PaletteEntry>();
    result.add(PaletteEntry.MANUAL_TASK);
    result.add(PaletteEntry.SCRIPT_TASK);
    return result;
  }

}
```
禁用上面两个扩展的结果就入下图所示。你可以看到，在 **Tasks**抽屉一栏，就看不到 *manual task*和*script task*。  
![禁用形状](/images/activiti/designer.palette.disable.manual.and.script.png)  
要想禁用所有默认的形状，你可以使用下面这样的代码：
```java
public class MyPaletteCustomizer extends AbstractDefaultPaletteCustomizer {

  /*
   * (non-Javadoc)
   *
   * @see org.activiti.designer.integration.palette.DefaultPaletteCustomizer#disablePaletteEntries()
   */
  @Override
  public List<PaletteEntry> disablePaletteEntries() {
    List<PaletteEntry> result = new ArrayList<PaletteEntry>();
    result.add(PaletteEntry.ALL);
    return result;
  }

}
```
结果如下图所示(注意，默认形状的抽屉也从面板中移除了):  
![禁用所有形状](/images/activiti/designer.palette.disable.all.png)
### 校验图表并导出为自定义输出格式
除了自定义工具面板，你还可以让Activiti设计器校验图表并将图表中的信息保存在Eclipse工作区中的自定义资源目录下。我们提供了该功能的内置扩展点，本节我们将会讲解如何使用它们。
> 最近我们重新引入了 ExportMarshaller 函数。我们还在研究校验功能。下文详细地介绍了我们之前如何做的，我们会在新功能可用的时候进行更新。

Activiti设计器允许你自行编写扩展程序来校验图表。默认情况下，该设计器已经内置了BPMN结构校验器，但是如果你想校验附加选项比如模型惯例或者 **CustomServiceTask**中的属性，你可以对它进行扩展。这些扩展就是人们熟知的 **Process Validators**。

你还可以扩展Activiti设计器，让它在保存图表时保存为其他格式。这种扩展称为 **Export Marshallers**，在用户点击保存的时候，由Activiti设计器自动执行。如果扩展能检测到某个扩展名，通过在Eclipse 的首选项对话框设置，就可以启动或者禁用此行为。设计器会根据用户的首选项设置，在你保存图表的时候，确保 **ExportMarshaller**的执行。

通常，你会想将 **ProcessValidator**和 **ExportMarshaller**结合起来。假设你有很多正在使用的 **CustomServiceTask**，它们包含你希望在生成的流程中使用的属性。但是，在流程生成之前，你希望校验它们中的一部分值。将**ProcessValidator**和 **ExportMarshaller**结合起来是达成这个效果的最好方式，而且Activiti设计器也允许你无缝接入你的扩展。

为了创建一个 **ProcessValidator** 或者 **ExportMarshaller**，你需要创建一个与扩展工具面板不同类型的扩展。原因很简单：在你的代码中，你需要访问比集成库提供的还多的API。尤其是，你需要Eclipse自身的类。所以，一开始，你应该创建一个Eclipse插件(你可以通过使用Eclipse对PDE的支持来完成)并将其打包到自定义的Eclipse产品或者功能中。解释开发Eclipse插件所涉及的所有细节超出了本文档的范围，因此下面我们只介绍如何扩展 Activiti 设计器的功能。

你的包需要依赖下面这些库：
- org.eclipse.core.runtime
- org.eclipse.core.resources
- org.activiti.designer.eclipse
- org.activiti.designer.libs
- org.activiti.designer.util

(可选)如果你想在扩展中使用 org.apache.commons.lang 包，可以通过设计器使用它。

**ProcessValidators**和**ExportMarshallers**都是通过扩展基类来创建的。这些基类从它们的父类**AbstractDiagramWorker**中继承了一些很有用的方法。使用这些方法，你可以创建信息，警告和错误标记，这些标记显示在Eclipse的 problems视图中，以供用户找出错误或者重要的信息。你可以以**Resources**和 **InputStreams** 的形式获取图表的信息。该信息由 **DiagramWorkerContext**提供，你可以从**AbstractDiagramWorker**类中获取到它。

在 **ProcessValidator**或者 **ExportMarshaller**中首先调用 **clearMarkers()**方法是一个好主意；它会清除你工程里之前所有的标记(标记自动链接到工程，清除一个工程的标记，对其他工程没有影响)。比如：
```java
// Clear markers for this diagram first
clearMarkersForDiagram();
```
你还应该使用 **DiagramWorkerContext**提供的进度监视器将进度报告给用户，因为验证和/或编组操作可能会占用用户的等待时间。报告进度需要了解如何使用Eclipse的功能。你可以点击 [本文](http://www.eclipse.org/articles/Article-Progress-Monitors/article.html)来获得有关概念和用法的详尽说明。
#### 创建一个流程校验器的扩展
> 正在审核中！

在 **plugin.xml** 文件中创建 **org.activiti.designer.eclipse.extension.validation.ProcessValidator**的扩展点。该扩展点需要继承 **AbstractProcessValidator** 类。
```xml
<?eclipse version="3.6"?>
<plugin>
  <extension
    point="org.activiti.designer.eclipse.extension.validation.ProcessValidator">
    <ProcessValidator
      class="org.acme.validation.AcmeProcessValidator">
    </ProcessValidator>
  </extension>
</plugin>
```
```xml
public class AcmeProcessValidator extends AbstractProcessValidator {
}
```
你需要实现一系列的方法。最重要的是 **getValidatorId()**方法，它你会为你的校验器返回一个全局唯一的ID。你可以通过 **ExportMarshaller**来执行它，或者甚至让 *其他人*通过他们的 **ExportMarshaller**来执行你的校验器。实现 **getValidatorName()**并返回你校验器的逻辑名。该名称会在对话框中展示给用户。在 **getFormatName()**中，你可以返回校验器正在校验的图表类型。

**validateDiagram()**方法中执行着校验工作。在这里，你可以使用你自己的代码实现特定功能了。但是，通常情况下，你需要先拿到图表流程中的节点，通过迭代它们来收集、比较和校验数据。下面的代码展示了如果做上述工作：
```java
final EList<EObject> contents = getResourceForDiagram(diagram).getContents();
for (final EObject object : contents) {
  if (object instanceof StartEvent ) {
  // Perform some validations for StartEvents
  }
  // Other node types and validations
}
```
在通过验证之后，别忘了执行 **addProblemToDiagram()** 和/或者 **addWarningToDiagram()**等。确保你返回了一个正确的boolean值来表明用户是否通过了该验证。可以通过执行 **ExportMarshaller**来决定下一步操作。

#### 创建一个 ExportMarshaller 扩展
在 **plugin.xml**中创建一个 **org.activiti.designer.eclipse.extension.ExportMarshaller**的扩展点。该扩展点需要继承 **AbstractExportMarshaller**类。该抽象基类在编组到你自己的格式中时，提供了一组很有用的方法，但是最重要的还是它允许你将资源保存到工作区并调用校验器。

设计器的源码文件夹中提供了一个示例实现。该示例展示了如何使用基类的这些方法来完成基本功能，比如访问图表的 **InputStream**，使用它的 **BpmnModel**以及将资源保存在工作区。
```xml
<?eclipse version="3.6"?>
<plugin>
  <extension
    point="org.activiti.designer.eclipse.extension.ExportMarshaller">
    <ExportMarshaller
      class="org.acme.export.AcmeExportMarshaller">
    </ExportMarshaller>
  </extension>
  </plugin>
  ```
  ```java
  public class AcmeExportMarshaller extends AbstractExportMarshaller {
}
```
你需要实现一些方法，比如 **getMarshallerName()**和 **getFormatName()**。这些方法用于向用户展示选项，并在进度框中展示信息，所以请确保你返回的描述能正确反映你正在实现的功能。

你的大部分工作是在 **doMarshallDiagram()**中做的。

如果你想先执行一个校验，可以在你的 marshaller 中直接调用校验器。你可以从该校验器中拿到一个boolean返回值，所以你知道该校验是否正确执行。在大多数情况下，如果图表不合法，你不会想去继续编组图表，但是如果校验失败，你可能会选择继续进行，甚至创建一个不同的资源。

获得所需要的所有数据后，你要调用 **saveResource()**方法来创建包含数据的文件。你可以根据需要，在单个 ExportMarshaller 中多次调用 **saveResource()**；因此，编组器可以用于创建多个输出文件。

你可以使用 **AbstractDiagramWorker**中的 **saveResource()**方法为输出资源构造文件名。你可以解析几个有用的变量来创建文件名，比如 _original-filename__my-format-name.xml。这些变量在Javadoc 的 ExportMarshaller接口中有描述。如果你还想自己解析占位符，可以对字符串（比如 path）使用 **resolvePlaceholders()**方法。**getURIRelativeToDiagram()** 将会调用该方法。

你应该使用提供的流程监视器将进度报告给用户。[该文章](http://www.eclipse.org/articles/Article-Progress-Monitors/article.html)介绍了如果使用它。