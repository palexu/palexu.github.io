---
title: 嵌套类与序列化问题
date: 2017-08-02 12:47:00
---

有时候开发时为了方便，在代码中使用了不少嵌套类(nested class)，但是使用过程中如果不了解嵌套类的特性，可能会造成意想不到的情况。

Java对象想要通过网络传输，必须实现序列化接口，于是我们在Service中定义了了一个实现了序列化接口的内部类，然后调用远程接口将其传输到网络上。
一运行报异常：不能将没有实现序列化接口的Object序列化。
怎么回事，这是一个很简单的内部类，的确已经实现了序列化接口了，其定义的成员都是可序列化的String类型;将其换成普通类没有问题。难道不能使用序列化的内部类?

其实我们使用的内部类是嵌套类(nested class)的一种，而nested class 共有四种：
   - static nested class 静态嵌套类
   - inner class 内部类(非静态)
   - local class 本地类(定义在方法内部)
   - anonymous class 匿名类

静态嵌套类的行为更接近普通的类，另外三个是真正的内部类。区别在于作用域的不同。

以下是对他们的性质描述：

以下还有一段程序说明它们的区别和使用方式：

```java
/**
*
* 普通内部类持有对外部类的一个引用， 静态内部类却没有
*
*/
public class OutterClass {

    /*
     * this is static nested class
     */
    private static class StaticNestedClass {
        private void yell() {
            System.out.println(this.toString());
            // OutterClass.this.yell();//静态内部类实例没有外部类实例的引用
        }
    }

    /*
     * this is inner class
     */
    private class InnerClass {
        private void yell() {
            System.out.println(this.toString());
            OutterClass.this.yell();//内部类实例显式使用外部类实例的方式
        }
    }

    private void yell() {
        System.out.println( this.toString());
    }

    private void run() {
        /*
         * this is local class
         */
        class LocalClass {
            public void yell(){
                System.out.println(this.toString());
            }
        }
        /*
         * this is anonymous class
         */
        new Object() {
            public void yell(){
                System.out.println(this.toString());
            }
        }.yell();
        LocalClass lc=new LocalClass();
        InnerClass ic = new InnerClass();
        StaticNestedClass sc=new StaticNestedClass();
        lc.yell();
        ic.yell();
        sc.yell();
    }

    public static void main(String[] args) {
        OutterClass oc = new OutterClass();
        oc.run();
    }
}
```
仔细分析如上代码，可以得出一个结论，所有的内部类，Local内部类，匿名内部类都可以直接访问外面的封装类的实例变量和方法。而静态嵌套类则不能。
调试代码可以发现，内部类，Local内部类，匿名内部类的实例都持有一个外部封装类实例的隐式引用;
而java对象序列化要求对象里所有的对象成员都实现序列化接口。
所以，如果只有内部类实现序列化，而外部封装类没有实现序列化接口，就会在对内部类进行序列化的时候报出异常。
