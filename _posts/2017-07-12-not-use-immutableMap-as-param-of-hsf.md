---
title: 不要使用Guava的ImmutableList、ImmutableMap 来传递hsf的参数
date: 2017-07-12 15:27:31
---
hsf的服务不能传guava的`ImmutableList`和`ImmutableMap`。

## 原因

```java
public static <E> ImmutableList<E> of(E element) {
   return new SingletonImmutableList<E>(element);
 }
```

```java
final class SingletonImmutableList<E> extends ImmutableList<E> {
  final transient E element;

  SingletonImmutableList(E element) {
    this.element = checkNotNull(element);
  }

  public E get(int index) {
    Preconditions.checkElementIndex(index, 1);
    return element;
  }
```

`transient`关键字表示相关字段不需要被序列化,但是hsf和dubbo是要先序列化然后再进行tcp传输的，因此参数就无法传递过去。

## 解决方案
可以用`java.util.Collections#singletonList, 因为 这个方法返回的`list、map、set`不会有`transient`字段。

---

## 相关
transient的用途

Q：transient关键字能实现什么？

A：当对象被序列化时（写入字节序列到目标文件）时，transient阻止实例中那些用此关键字声明的变量持久化；当对象被反序列化时（从源文件读取字节序列进行重构），这样的实例变量值不会被持久化和恢复。例如，当反序列化对象——数据流（例如，文件）可能不存在时，原因是你的对象中存在类型为java.io.InputStream的变量，序列化时这些变量引用的输入流无法被打开。
