---
author: qunxi
create: 2017-03-15 18:43+08:00
update: 2017-03-25 18:43+08:00
layout: page
title: "设计模式-Adapter模式"
description: ""
comments : true
categories:
tags:
- 设计
- 模式
---

# 前言

除了前面介绍的创建型模式，在面向对象设计中，利用对象继承或组合来扩展对象结构也是非常常见的方式，于是我们总结了很多关于复用结构的模式，比如*Adapter模式*、*Bridge模式*、*Decorator模式*等结构型模式。本篇文章我们将介绍最常见的结构型模式-**Adapter模式**。
<!--more-->

# 1. 接口的转换

在现实生活中，我们不得不承认使用适配器的场景随处可以见，因为时地域，时间，利益等诸多因素的影响，我们不可能指定一个唯一的标准让所有人来遵守。所以在进行不同标准交互时，我们必须设计一些中间件（适配器）来完成功能的转换。从小到各种电器的转接口，大到翻译人员，其实都是Adapter模式的应用，在代码的世界里亦是如此。下面我们先来了解下设计模式中对Adapter模式的定义：

> 将一个类的接口转换成客户希望的另外一个接口。 Adapter模式使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。

虽然定义已经非常清晰，但是我依然希望从一个具体的例子开始，这样便于描述和理解。假设公司平台组实现了一个新的通用库，它支持大部分的通用数据结构和常用算法，其中有一个叫双向队列的容器，它可以在队列的首尾插入或提取元素。因为它的灵活性，我们可以通过它完成先进先出（队列）或后进先出（栈）的功能。下面就是其主要接口和实现类：

```
interface IDeque<T>{
    void PushBack(T value);
    void PushFront(T value);
    T PopBack();
    T PopFront();
    int Size(); 
}

class Deque<T> : IDeque<T>{
    public void PushBack(T value){
    }
    
    public void PushBack(T value){
    }

    public void PushFront(T value){
    }
    
    public T PopBack(){
    }

    public T PopFront(){
    }

    public  int Size(){
    }
}
```

为了代码和设计的一致性，我们决定将所代码都升级支持新的通用库。可是在已有的项目中我们对于栈（Stack）的使用已经定义了一个`IStack`接口，虽然我们可以通过`Deque`来完成栈类是的功能，但是我们必须修改很多已有的接口和涉及所有接口变化的客户代码。这样的改变不仅成本高，而且也违背了**迪米特**和**单一职责**原则。下面是之前客户代码定义的栈接口。

```
interface IStack<T>{
    void Push(T value);
    void Pop();
    int Size();
}
```

# 2. Adapter模式-继承实现

## 继承之Is-a

为了适配客户代码认同的接口`IStack`，我们必须实现`IStack`的接口，但是也没有必要从新造一次轮子。所以如何复用`Deque`的已有功能呢？作为面向对象语言，我们最先想到的一定是继承。下面便是利用继承的实现方式：

```
class Stack<T> : Deque<T>, IStack<T>{
    public void Push(T value){
        base.PushBack(value);
    }

    public void Pop(){
        base.PopBack();
    }

    public int Size(){
        return base.Size();
    }
}
```
因为使用了继承，所以`Stack`拥有了`Deque`所有的特性和行为，但是这也带来了不好的副作用，比如`PopFront`、`PushFront`等本来不属于栈的行为也被包含到`Stack`中。Java, C#我们不能通过继承完全的避免这个问题，所以在这些语言中我们一般不建议使用继承的模式实现Adapter模式。但是C++因为语言特性的支持，我们可以更好的为`Stack`做到封装。

## 继承之Has-a

C++语言为继承提供了三种限制（`public`、`protected`、`private`），其中`public`和C#、Java的继承是一样的。但是`protected`、`private`却可以进一步限制基类函数以及属性的访问权限。比如`private`继承会使所有基类的`public`函数或属性变成私有的。这样父类的公共函数或属性只能在继承类被访问，而不能被客户代码访问。所以私有继承已经打破了传统继承关系(is-a)，基类更像是封装后辅助类。下面是C++利用私有继承的实现方式:

```
// C++语言本身并不直接提供接口概念，但是我们可以通过纯虚函数实现接口
class IStack<T>{
    public:
        virtual void Push(T value) = 0;
        virtual void Pop() = 0;
        virtual int Size() = 0;
}

class Stack<T> : private Deque<T>, public IStack<T>{
    public:
        void Push(T value){
            Deque<T>::PushBack(value);
        }

        void Pop(){
            Deque<T>::Pop();
        }

        void int Size(){
            return Deque<T>::Size();
        }
}
```

通过私有继承的封装，客户代码再也不可以通过Stack访问到Deque的任何方法。下面是Adpater模式的类图结构：
![builder pattern](\post-images\2017_3_25_class_adapter_pattern.png)


虽然已继承的方式可以快熟的实现适配的功能。但是它却引入了继承本身带来的缺点，比如：

* 不能完全封装被适配对象（Adaptee），比如像C#、Java这样不提供继承限制的语言。

* 适配器对象（Adapter）依赖于被适配对象（Adaptee），被适配对象一旦发生变化就会直接影响适配对象。

* 如果被适配对象（Adaptee）本身就是一个庞大或继承关系复杂的对象，那么通过继承，适配类（Adapter）就会变的更加复杂，甚至类膨胀。


# 3. Adapter模式-组合实现

为了解决继承带来的缺点，尤其像C#或Java这种不支持继承限制的语言。其实**has-a**还有另外一种实现方式，那就是利用组合，将被适配对象（Adaptee）作为适配对象（Adapter）的成员，这样适配对象只会集成需要的功能，而且不是所有被适配对象。

```
class Stack<T> : IStack<T>{
    
    private Deque<T> container = new Deque<T>();

    public void Push(T value){
        base.PushBack(value);
    }

    public void Pop(){
        base.PopBack();
    }

    public int Size(){
        return base.Size();
    }
     
}
```
这种实现不仅和C++私有继承有异曲同功之处，而且也适合C#、Java等语言，其次是组合在封装上的优势明显强于继承，我们在[代码设计](https://qunxi.github.io/2017/01/30/object-oriented-design.html)这篇文章中有提到，这里我们不再赘述。下面是组合实现方式的类图：
![builder pattern](\post-images\2017_3_25_object_adapter_pattern.png)
有时我们也将Adapter模式叫做*Wrapper模式*，所以在语义上组合更接近Wrapper的意思。

# 后记

就像前面Stack的例子，其实我是在模仿STL（C++标准库）库的实现，它可以很好的复用已有数据结构，实现另一种数据结构。但是很多时候，我们也将Adapter模式作为一种亡羊补牢的方案，比如双方代码都不容易被修改的时候，使用Adapter模式可以很好的完成适配的功能。

## （转载本站文章请注明[作者和出处](https://qunxi.github.io/)，请勿用于任何商业用途）
