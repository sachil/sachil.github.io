---
layout: post
title:  Java基础知识
mathjax: false
date: 2020-08-31 20:34:23
urlname:  Java基础知识
categories:
  - Java
tags:
 - Java
---

# Java基础知识

## JVM

### 1.内存模型

`jvm`的内存模型可以分为两类：1.**线程私有的数据区域**，包括：程序计数器，虚拟机栈，本地方法栈。2.**所有线程共享的数据区域**，包括：方法区(常量池)，堆区。

- **程序计数器**，它是一块较小的内存，可以看作是当前线程所执行的字节码的行号指示器，字节码解释器在工作时就是通过改变它的值来选取下一条需要执行的字节码指令，所以每个线程都需要一个独立的程序计数器。
- **虚拟机栈**，它也是线程私有的，它的生命周期和线程相同，**用于存储局部变量，操作数栈，动态连接，方法出口**
- **本地方法栈**，线程私有，作用和虚拟机栈相似，不同的就是：虚拟机栈为虚拟机执行Java方法，而本地方法栈为虚拟机执行Native方法。
- **方法区(常量池)**，所有线程共享，用于存放已被虚拟机加载的类信息，常量，静态变量等数据，常量池是方法区的一部分，用于存放编译器生成的各种字面量和符号引用。
- **堆区**，所有线程共享，是Java虚拟机所管理的内存中最大的一块，用于存放对象的实例，几乎所有对象实例都在这里分配内存，也是垃圾收集器管理的主要区域

## 泛型

### 1.泛型支持协变吗，为什么？

**泛型是不支持协变的**，其原因可以通过举一个例子来说明：

```Java
class A{}
class B extends A{}
class C extends A{}

//定义两个List：list1和list2
List<A> list1 = new ArrayList<A>();
List<B> list2 = new ArrayList<B>();
//如果泛型支持协变，那么我们就可以这样赋值
//list1 = list2;
//也可以这样操作,因为list1可以持有A类型及其子类的实例，C类型就是A类型的子类；
//这样操作之后，通过list1的引用，间接的向list2中添加了C类型的实例，这样list2中
//也持有了C类型的实例，但是这与定义list2时，list2只能持有B类型及其子类的情况
//发生了冲突（C类型并不是B类型的子类），所以list2=list2是不合法的。
//list1.add(new C());
//同样的也不能这样赋值
//list2 = list1;
```

所以通过列举的例子可以看出，泛型是不能支持协变的,这会造成发生冲突的风险。

### 2.泛型中的通配符

泛型中通配符机制的目的是：**让一个持有特定类型(例如A类型)的集合，可以强制转换为持有该类型的子类或者父类(A类型的子类或者父类)的集合。**泛型通配符通常三种表达方式：

- 无界通配符\<?>，例如：`List<?> list;`，表示list是持有某种特定类型的List，该特定类型没有任何限制，但一定是特定的一种类型，因为不知道具体的类型，所以不能不能向其添加具体的实例(包括Object实例)，因为这是不安全的。而`List list;`这样不传入类型参数，表示list持有的是Object类型的元素，可以添加任何类型的实例，但是编译器会发生警告。

- 上界通配符<? extends T>，例如：`List<? extends Number> list;` ,表示list可以引用一个List及其子类的List对象，这个List对象持有的元素类型是Number及其子类。**使用上界通配符之后，list将只能读而不能写**，因为我们不知道list持有的元素的具体类型，所以我们不能向其中添加具体的实例，但是我们知道list中持有元素的类型是Number及其子类，所以我们可以将元素取出来当做Number来处理。

- 下界通配符<? super T>，例如：`List<? super Number> list;`,表示list可以引用一个List及其子类的List对象，这个List对象持有的元素类型是Number及其父类。**使用下界通配符之后，list可以添加Number及其子类的实例，但是不能添加Number的父类的实例，也可以将元素当做Object读取出来(Object肯定是其中所有元素的共同父类)**，因为我们不知道list持有的元素的具体类型，我们只知道元素的类型是Number及其父类，因为该元素的类型必定是Number及其子类的父类，所以我们可以向其中添加Number及其子类的实例，这样是安全的。而添加Number的父类的实例则是不安全的。

  泛型中通配符的用处是：

  1. 从一个泛型集合中读取元素(使用上界通配符)
  2. 向一个泛型集合中添加元素(使用下界通配符)

## 数组

### 1.数组的协变，以及潜在的风险

数组可以支持协变，但是有人认为这是一个缺陷，例如：

```java
public class Test {
    private class Fruit{}
    private class Apple extends Fruit{}
    private class LittleApple extends Apple{}
    private class Orange extends Fruit{}

    public void main(String[] args){
        //Fruit类是Apple的父类，所以这样赋值是可行的，一个Apple的数组也是一种Fruit的数组，这个就是数组的协变。
        Fruit[] fruits = new Apple[5];
        //可以通过编译，但是不可以运行，因为运行时，fruits实际是Apple类型的数组，只能存放Apple类及其子类。
        fruits[0] = new Fruit();
        //可以通过编译，但是不可以运行，原因同上。
        fruits[1] = new Orange();
        //运行正常。
        fruits[2] = new Apple();
        //运行正常。
        fruits[3] = new LittleApple();
    }
}
```

