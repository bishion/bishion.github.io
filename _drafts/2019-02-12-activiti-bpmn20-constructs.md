---
layout: post
title: Activiti 6.0 用户文档 - 第八章：BPMN2.0 规范
categories: activiti
description: 流程引擎 Activiti 第八章：BPMN2.0 规范
keywords: 流程引擎, activiti, BPMN
---
# BPMN2.0 规范
本章介绍 Activiti 支持的 BPMN2.0规范，以及对BPMN标准的自定义扩展。

## 自定义扩展
BPMN2.0 标准对于各个相关方来说都是好事。用户不用再依赖某个供应商的专有解决方案。像Activiti 这样的开源框架，可以实现与大型供应商相同(通常是更好的)的功能。有了 BPMN2.0标准，从供应商的解决方案可以平滑过渡到Activiti。

然而，标准的缺点在于，它通常是不同公司讨论和妥协的结果（通常是愿景）。开发在读流程定义的 BPMN2.0 XML 文档时，经常会觉得它太笨重了。鉴于 Activiti 是将易用性作为最高优先级，我们引入了 ***Activiti BPMN 扩展***。这些扩展采用了BMPN2.0标准之外的新方式来简化某些结构。

虽然BPMN2.0规范明确指出，它是为自定义扩展而制定的，但是我们还是要保证：
- 自定义扩展的先决条件是 **大多数情况下** 它能简化 **做事的基本方式**。所以，在使用自定义扩展的时候，你不用担心无法回头。
- 当使用一个扩展时，通常它会由一个新的XML标签、元素、属性来标识。比如，使用 **activiti：** 命名空间前缀

所以，你可以自由选择是否使用自定义扩展。一些现实情况会影响你做这些节点：图形化编辑界面的使用，公司的规章制度等。只有我们觉得原有标准中的一些地方有更简单和高效的实现的时候，我们才会去扩展它们。对于这些扩展，你可以尽管将使用体验反馈给我们（无论好评还是差评），也可以提供一些新的思路。谁知道呢，说不定有一天你的想法就称为标准了！

## 事件
事件是表示在流程处理过程中发生的一些事情。事件通常被可视化为一个圆圈。在BPMN2.0中有两个事件分类：*捕获*事件和*抛出*事件。
- 捕获：当流程执行到事件里，它就会等待触发。触发器的类型通过内置的图标或者XML中定义的类型声明。捕获事件跟抛出事件在可视化图表里可以很容易区分，因为捕获事件是一个未填充的圆(即白色)
- 抛出：当流程执行到事件里，它就会激活触发器。触发器的类型通过内置的图标或者XML中定义的类型声明。抛出事件跟捕获事件在可视化界面里可以很容易区分，因为抛出事件是一个黑色填充的圆。

### 事件定义
事件定义表示事件的语义。如果没有事件定义，事件就没什么特别的。比如，一个没有事件定义的开始事件无法指定如何启动流程。如果我们给开始事件添加一个流程定义(比如一个计时器事件定义)，我们就明确了什么类型的事件启动这个流程(比如计时器事件可以在某个特定时间触发)。

### 计时器事件定义
计时器事件是指通过计时器触发的事件。它可以用在 [启动事件](https://www.activiti.org/userguide/#bpmnTimerStartEvent),[中间事件](https://www.activiti.org/userguide/#bpmnIntermediateCatchingEvent)或者[边界事件](https://www.activiti.org/userguide/#bpmnTimerBoundaryEvent)。时间事件的行为取决于使用的业务日历。每个计时器事件都有一个默认的业务日历，但是我们也可以在计时器事件定义中定义业务日历。

```xml
<timerEventDefinition activiti:businessCalendarName="custom">
    ...
</timerEventDefinition>
```

*businessCalendarName*指向的是流程引擎中配置的业务日历。如果业务日历被省略了，系统就会启用默认的业务日历。

计时器定义必须是下面的元素的一种：
- timeDate：该格式指定了一个 [ISO 8601](http://en.wikipedia.org/wiki/ISO_8601#Dates)格式的固定日期，到了这个时间点，就会激活触发器。比如：
```xml
<timerEventDefinition>
    <timeDate>2011-03-11T12:13:14</timeDate>
</timerEventDefinition>
```
- timeDuration：我们可以给 *timerEventDefinition* 添加一个子元素 *timeDuration*，来指定计时器在触发之前应该运行多长时间。它使用 [ISO 8601](http://en.wikipedia.org/wiki/ISO_8601#Durations)格式。比如(间隔持续10天)：
```xml
<timerEventDefinition>
    <timeDuration>P10D</timeDuration>
</timerEventDefinition>
```
- timeCycle：指定重复间隔，这在定期启动流程或者给超时用户任务发送提醒场景下特别有用。timeCycle 元素有两种格式。一种是 [ISO 8601](http://en.wikipedia.org/wiki/ISO_8601#Repeating_intervals)标准指定的循环持续时间的格式。比如(重复3次，每次持续10小时)：

我们还可以给 *timeCycle*元素指定一个可选的属性 *endDate*，或者在时间表达式后面加上下面这句：**R3/PT10H/\${EndDate}**。当 endDate 时间到来时，系统将会停止创建任务。它既可以接收诸如 “2015-02-25T16:42:11+00:00” [ISO 8601]格式的静态值，也可以接收\${EndDate} 变量
```xml
<timerEventDefinition>
    <timeCycle activiti:endDate="2015-02-25T16:42:11+00:00">R3/PT10H</timeCycle>
</timerEventDefinition>
```
```xml
<timerEventDefinition>
    <timeCycle>R3/PT10H/${EndDate}</timeCycle>
</timerEventDefinition>
```
如果两种方式都定义了，系统将会使用endDate属性定义的值。  
现在只有 *BoundaryTimerEvents* 和 *CatchTimerEvent* 支持 *EndDate* 功能。  
另外，你还可以使用 cron 表达式来指定时间周期，下面例子表示每5分钟触发一次：
```properties
0 0/5 * * * ?
```
关于 cron 表达式，你可以参考[cron 教程](http://www.quartz-scheduler.org/docs/tutorials/crontrigger.html)。

**注意：**第一个字符表示秒，并不是正常的 Unix cron 中的分钟。

循环持续时间更适合处理相对定时器，相对定时器是根据某个特定时间点(比如，启动用户任务的时间)计算的，而 cron 表达式可以处理绝对定时器-这对于 [定时器启动事件](https://www.activiti.org/userguide/#timerStartEventDescription)特别有用。

你可以使用表达式来定义计时器事件，这样可以将流程变量放在计时器定义中。该流程变量必须给适当的计时器类型传入 ISO 8601(或者 cron 表达式) 字符串。
```xml
<boundaryEvent id="escalationTimer" cancelActivity="true" attachedToRef="firstLineSupport">
  <timerEventDefinition>
    <timeDuration>${duration}</timeDuration>
  </timerEventDefinition>
</boundaryEvent>
```
**注意：**只有在任务或者异步执行器被启动的时候才能触发计时器(即：在 **activiti.cfg.xml** 中将 *jobExecutorActivate*或*asyncExecutorActivate* 设置为 **true**，因为默认情况下它们是禁用状态)

### 错误事件定义
**重要提示：**BPMN 中的错误 **不是** java 中的异常。实际上，它俩没有任何相同之处。BPMN 错误是一种对 *业务异常* 的建模方式。Java 异常有它自己的[处理方式](https://www.activiti.org/userguide/#serviceTaskExceptionHandling)。
```xml
<endEvent id="myErrorEndEvent">
  <errorEventDefinition errorRef="myError" />
</endEvent>
```
### 信号事件定义
信号事件是引用一个信号的事件。信号是指一个全局的事件(比如广播)被传递给所有活动的处理器(正在等待流程实例/捕获信号事件)。

信号事件通过 **signalEventDefinition** 元素声明。**signalRef** 指向 **signal**元素，该元素在根元素 **definitions** 下面定义。下面是一个流程的片段，它定义了一个信号事件被抛出，然后中间的事件捕获。

```xml
<definitions... >
	<!-- declaration of the signal -->
	<signal id="alertSignal" name="alert" />

	<process id="catchSignal">
		<intermediateThrowEvent id="throwSignalEvent" name="Alert">
			<!-- signal event definition -->
			<signalEventDefinition signalRef="alertSignal" />
		</intermediateThrowEvent>
		...
		<intermediateCatchEvent id="catchSignalEvent" name="On Alert">
			<!-- signal event definition -->
			<signalEventDefinition signalRef="alertSignal" />
		</intermediateCatchEvent>
		...
	</process>
</definitions>
```
**signalEventDefinition** 引用了相同的相同的 **signal** 元素。

#### 抛出信号事件
一个信号既可以被 BPMN 结构的流程实例抛出，又可以被 java API 抛出。**org.activiti.engine.RuntimeService** 中的下面两个方法可以用来抛出信号：
```java
RuntimeService.signalEventReceived(String signalName);
RuntimeService.signalEventReceived(String signalName, String executionId);
```

