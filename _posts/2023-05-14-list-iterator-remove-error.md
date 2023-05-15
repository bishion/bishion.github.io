---
layout: post
title: 迭代过程中移除list元素不一定会报错
categories: diary
description: 在迭代过程中，移除list中倒数第二个元素不会报错
keywords: list, iterator, remove, ConcurrentModificationException
---
# 迭代过程中移除list元素不一定会报错
记忆中，在 foreach 循环期间，调用 *list.remove()* 方法时，会报 ConcurrentModificationException 异常。但是，一个偶然的机会，我发现，如果要移除的元素位于倒数第二个时，程序并不会抛异常，且能正常运行。

## 背景
使用 foreach 循环一个 list, 并在循环期间移除指定元素时，会抛异常:
```java
 public static void main(String[] args) {
    List<Integer> list = new ArrayList<Integer>(4);
    list.add(1);
    list.add(2);
    list.add(3);
    list.add(4);

    for (Integer value : list) {
        if (value.equals(2)) {
            list.remove(value);
        }
    }
    System.out.println(list);
 }
```
抛出异常如下:
```log
Exception in thread "main" java.util.ConcurrentModificationException
	at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:911)
	at java.util.ArrayList$Itr.next(ArrayList.java:861)
	at xx.xx.xx.ForeachDemoTest.main(ForeachDemoTest.java:xx)

```
### 分析
首先，foreach 循环本身是一个语法糖，我们将该段代码编译后并反编译，就会看到原始的写法如下：
```java
public static void main(String[] args) {
    List<Integer> list = new ArrayList(4);
    list.add(1);
    list.add(2);
    list.add(3);
    list.add(4);
    Iterator iterator = list.iterator();

    while(iterator.hasNext()) {
        Integer value = (Integer)iterator.next();
        if (value.equals(2)) {
            list.remove(value);
        }
    }

    System.out.println(list);
}
```
异常原因很简单，本身 ArrayList 是非线程安全的，为了防止 list 在迭代过程中，被另外一个线程修改了元素，造成未知风险，所以在迭代过程中增加了对元素的数量校验。这也是为什么，报错行并不在 *list.remove()*, 而是在 foreach 循环上，确切的说，是在 *iterator.next()* 上。这里，我们也看下报错的代码:
```java
final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```
这里的 modCount 就是在 *AbstractList* 中，专门给它的实现类用来做快速失败用的:
```java
/**
The number of times this list has been structurally modified. Structural modifications are those that change the size of the list, or otherwise perturb it in such a fashion that iterations in progress may yield incorrect results.
This field is used by the iterator and list iterator implementation returned by the iterator and listIterator methods. If the value of this field changes unexpectedly, the iterator (or list iterator) will throw a ConcurrentModificationException in response to the next, remove, previous, set or add operations. This provides fail-fast behavior, rather than non-deterministic behavior in the face of concurrent modification during iteration.
Use of this field by subclasses is optional. If a subclass wishes to provide fail-fast iterators (and list iterators), then it merely has to increment this field in its add(int, E) and remove(int) methods (and any other methods that it overrides that result in structural modifications to the list). A single call to add(int, E) or remove(int) must add no more than one to this field, or the iterators (and list iterators) will throw bogus ConcurrentModificationExceptions. If an implementation does not wish to provide fail-fast iterators, this field may be ignored
*/
/**
 * modCount 表示当前 list 结构发生改变的次数。 结构发生改变，是指 list 长度改变，或者 list 顺序被打乱等会导致 list 在迭代过程中出错的变化。
 */
protected transient int modCount = 0;
```
### 报错原因总结
因为 list 内置了检查机制，即在迭代过程中通过 modCount 检查 list 是否发生变化。如果 list 发生了变化，则在迭代就会报错。

## 移除倒数第二个元素不会报错
既然如上文所说，在迭代过程中修改 list 会报错，那么为何移除倒数第二个元素反而能正常运行呢？
### 分析
如果跟踪代码，我们会发现，当移除倒数第二个元素后，循环条件中的 *iterator.hasNext()* 会直接返回false。也就是说，当移除倒数第二个元素后，循环就终止了。这是因为，*hasNext()* 并不会检查 modCount, 只是单纯地比较了当前的迭代索引是否跟 list 长度相等；而报*ConcurrentModificationException* 是在 *next()* 方法中调用 *checkForComodification()* 抛出的：
```java
public boolean hasNext() {
    return cursor != size;
}

public E next() {
    checkForComodification();
    。。。。
}
```
此时，以值为 [1,2,3,4] 的 list 为例，当移除元素 *3* 时，执行过程是这样的：
1. 游标走到 2, 此时 size = 4
2. 判断当前元素值为 3, list 执行 remove(), 此时 size = 3
3. 当前循环体执行完毕，游标后移，此时 cursor = 3
4. cursor != size 值为 false, 循环结束

### 结论
移除倒数第二个元素后，因迭代循环直接结束，不会触发 modCount 校验逻辑，进而不会报错。

## 使用迭代器的 *remove* 不会报错
迭代器中的 *remove()* 代码如下：
```java
public void remove() {
    if (lastRet < 0)
        throw new IllegalStateException();
    checkForComodification();

    try {
        ArrayList.this.remove(lastRet);
        cursor = lastRet;
        lastRet = -1;
        expectedModCount = modCount;
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
```
### 分析
迭代器中的移除方法考虑到下标校验的问题，它在调用 *remove()* 后，手动将下标回退，并通过 *expectedModCount = modCount* 强行绕过 *checkForComodification()* 的检查

## 总结
1. 以前认为 *foreach* 时做 *remove()* 会报错，迭代器不会；现在来看，*foreach* 本质上也是迭代器。使用迭代器循环时，再使用 *list.remove()*, 就会报错
2. 在循环中，调用 *list.remove(Object)* 挺傻的，因为 *list.remove(Object)* 时间复杂度也是O(n)
3. 在链表中，调用 *list.remove(index)* 也挺傻的，理由同上
4. 回到开头，移除 list 中的指定元素，有如下几种方式

```java
/**
 * 使用 for 循环, 防止下标跳跃, 可以在移除后，下标后退
 */
public static List<Integer> removeTarget1(List<Integer> list, Integer target) {
    for (int i = 0; i < list.size(); i++) {
        if (list.get(i).equals(target)) {
            list.remove(i);
            i--;
        }
    }
    return list;
}
/**
 * 使用 for 循环, 防止下标跳跃, 可以从 list 尾部往前循环
 */
public static List<Integer> removeTarget2(List<Integer> list, Integer target) {
    for (int i = list.size()-1; i >= 0; i--) {
        if (list.get(i).equals(target)) {
            list.remove(i);
        }
    }
    return list;
}
/**
 * 使用迭代器, 注意别在迭代中又画蛇添足使用了 list.remove()
 */
public static List<Integer> removeTarget3(List<Integer> list, Integer target) {
    Iterator<Integer> iterator = list.iterator();
    while (iterator.hasNext()) {
        Integer value = iterator.next();
        if (value.equals(target)) {
            iterator.remove();
        }
    }
    return list;
}
/**
 * 使用系统自带方法(它内部也是使用了迭代器，基本上跟 removeTarget3 一样)
 */
public static List<Integer> removeTarget4(List<Integer> list, final Integer target) {
    list.removeIf(item -> item.equals(target));
    return list;
}

```
