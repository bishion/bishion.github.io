---
layout: wiki
title: 软件设计 6 原则
categories: 软件设计
description: 软件设计 6 原则
keywords: 设计模式, 软件设计
---
# SOLID
> In object-oriented computer programming, the term SOLID is a mnemonic acronym for five design principles intended to make software designs more understandable, flexible and maintainable.

> 翻译：在面向对象编程领域，SOLID 是五种设计准则的缩写。依照这些设计准则，软件在可读性、扩展性和可维护性方面变得更好 --维基百科

# 六种设计准则
现在公认的软件设计准则有六个，其实还是可以简写成 SOLID

- Single Responsibility
- Open/closed
- Liskov substitution
- Law of Demeter / Least Knowledge Principle
- Interface segregation
- Dependency inversion

## 单一职责原则，Single Responsibility
> The single responsibility principle is a computer programming principle that states that every module or class should have responsibility over a single part of the functionality provided by the software, and that responsibility should be entirely encapsulated by the class.
All its services should be narrowly aligned with that responsibility. Robert C. Martin expresses the principle as, "A class should have only one reason to change." --wikipedia
> 翻译：单一职责原则是指：每个模块或者类应该只对软件的单一功能负责，并且该职责完全由类来封装。
它的功能应该跟职责完全保持一致(不能多做也不能少做)。Robert C. Martin 表述这个原则为：Class 变更的原因只能是一个。 --维基百科

## 开闭原则，Open/closed
> In object-oriented programming, the open/closed principle states "software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification";
that is, such an entity can allow its behaviour to be extended without modifying its source code. --wikipedia

> 翻译：在面向对象编程领域，开闭原则规定『软件实体(类，模块，方法等等)必须对扩展是开放的，但是对修改是关闭的』。
就是说，你可以通过不修改它原来的代码来扩展它的功能

## 里氏替换原则，Liskov substitution
> Substitutability is a principle in object-oriented programming stating that, in a computer program, if S is a subtype of T, then objects of type T may be replaced with objects of type S (i.e. an object of type T may be substituted with any object of a subtype S) without altering any of the desirable properties of the program (correctness, task performed, etc.).
More formally, the Liskov substitution principle (LSP) is a particular definition of a subtyping relation, called (strong) behavioral subtyping, that was initially introduced by Barbara Liskov in a 1987 conference keynote address titled Data abstraction and hierarchy.
It is a semantic rather than merely syntactic relation; because, it intends to guarantee semantic interoperability of types in a hierarchy, object types in particular. --wikipedia

> 可替换原则是面向对象编程中的一个原则，表述为：在程序中，如果 S 是 T 的子类型，那么对象 T 可以被对象 S 替换(即 T 的对象可以被 S 的对象取代)而不会改变程序的任何期望属性(正确性，任务正常执行等)。
更正式地说，Liskov 替代原则(LSP)是子类型的特殊定义，称为(强)行为子类型，最初是由 Barbara Liskov 在 1987年的一次会议中提出，演讲主题是《数据抽象和层次》。
它不仅仅是句法关系，更是一种语义；因为它意在保证在层次结构类型中语义的互通性，特别是对象类型。 --维基百科

## 迪米特法则，Law of Demeter / Least Knowledge Principle
> The Law of Demeter (LoD) or principle of least knowledge is a design guideline for developing software, particularly object-oriented programs.
In its general form, the LoD is a specific case of loose coupling.
The guideline was proposed by Ian Holland at Northeastern University towards the end of 1987, and can be succinctly summarized in each of the following ways:
> - Each unit should have only limited knowledge about other units: only units "closely" related to the current unit.
> - Each unit should only talk to its friends; don't talk to strangers. --wikipedia
> - Only talk to your immediate friends.

> 翻译：迪米特法则(LoD)或者最少知道原则是一个软件开发设计准则，尤其是在面向对象编程领域。一般来说，迪米特法则是松散耦合的特殊情况。
这个准则是1987年底由 Ian Holland 在波士顿东北大学提出来的。它可以有以下几种方式表述：
> - 每个模块只限定于了解跟自己有密切关系的模块
> - 每个模块只跟自己的朋友说话，不跟陌生人说话
> - 只跟自己的朋友说话 --维基百科

## 接口隔离原则，Interface segregation
>The interface-segregation principle (ISP) states that no client should be forced to depend on methods it does not use.
ISP splits interfaces that are very large into smaller and more specific ones so that clients will only have to know about the methods that are of interest to them.
Such shrunken interfaces are also called role interfaces. ISP is intended to keep a system decoupled and thus easier to refactor, change, and redeploy. --wikipedia

> 翻译：接口隔离原则(ISP)规定，不能强迫客户端依赖它用不到的方法。
ISP 将大接口分割成小而具体的接口，这样客户端就只需知道它感兴趣的方法。这种小接口也叫角色接口。ISP 旨在保持系统解耦从而更容易重构、更改和重新发布。 --维基百科

## 依赖倒置原则，Dependency inversion
> In object-oriented design, the dependency inversion principle refers to a specific form of decoupling software modules.
When following this principle, the conventional dependency relationships established from high-level, policy-setting modules to low-level, dependency modules are reversed, thus rendering high-level modules independent of the low-level module implementation details.
The principle states:
> - High-level modules should not depend on low-level modules. Both should depend on abstractions.
> - Abstractions should not depend on details. Details should depend on abstractions.

> 翻译：在面向对象设计中，依赖倒置原则指的是软件模块解耦的特殊形式。
按照这个准则，上层模块依赖底层模块的传统依赖关系被颠倒过来，从而使得上层模块独立于下层模块的实现细节。该原则规定：
> - 上层模块不能依赖下层模块。他们应该同时依赖于抽象。
> - 抽象不应该依赖细节。细节应该取决于抽象。--维基百科
