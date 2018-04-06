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
All its services should be narrowly aligned with that responsibility. Robert C. Martin expresses the principle as, "A class should have only one reason to change."

> 翻译：单一职责原则是指：每个模块或者类应该只对软件的单一功能负责，并且该职责完全由类来封装。
它的功能应该跟职责完全保持一致(不能多做也不能少做)。Robert C. Martin 表述这个原则为：Class 变更的原因只能是一个。 --维基百科

## 开闭原则，Open/closed
> In object-oriented programming, the open/closed principle states "software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification";
that is, such an entity can allow its behaviour to be extended without modifying its source code.

> 翻译：在面向对象编程领域，开闭原则规定"软件实体(类，模块，方法等等)必须对扩展是开放的，但是对修改是关闭的。
就是说，你可以通过不修改它原来的代码来扩展它的功能

## 里氏替换原则，Liskov substitution
> Substitutability is a principle in object-oriented programming stating that, in a computer program, if S is a subtype of T, then objects of type T may be replaced with objects of type S (i.e. an object of type T may be substituted with any object of a subtype S) without altering any of the desirable properties of the program (correctness, task performed, etc.). More formally, the Liskov substitution principle (LSP) is a particular definition of a subtyping relation, called (strong) behavioral subtyping, that was initially introduced by Barbara Liskov in a 1987 conference keynote address titled Data abstraction and hierarchy. It is a semantic rather than merely syntactic relation; because, it intends to guarantee semantic interoperability of types in a hierarchy, object types in particular.

> 可替换原则是面向对象编程中的一个原则，表述为：在程序中，如果 S 是 T 的子类型，那么对象 T 可以被对象 S 替换(即)