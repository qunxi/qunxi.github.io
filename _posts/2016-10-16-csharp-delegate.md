---
author: qunxi
create: 2016-10-16 18:43+08:00
update: 2016-10-16 18:43+08:00
layout: page
title: "C#委托(Delegate)"
description: ""
comments : true
categories:
tags:
- 基础
---

# 1. 回调函数
在介绍delegate之前有必要了解一个概念叫回调函数，回调函数往往以一个函数作为参数被另一个函数所调用。我们知道在C++中有函数指针，甚至仿函数(Functor)来作为函数参数，javascript更是大行其道的使用回调函数(callback hell)，函数式编程那就更不用提了。那为什么需要回调函数呢？
<!--more-->

>## 关注点分离(Separation of concerns)

一个好的函数应该只封装本身所关注行为，比如一个快速排序算法，它的职责是实现快速排序本身的逻辑，至于排序时的比较条件(比如数组中的对象以什么规则比较大小)应该是有外部客户决定。下面的Predicate操作就是一个回调函数

```C#
	void sort (RandomAccessIterator first, RandomAccessIterator last, Predicate comp);
```

>## 步调用(Asynchronization)

很多时候为了减少因大量的IO操作带来不好的用户体验，我们往往会使用异步处理。其中AJAX的出现给互联网带来全新的体验，所有的网络请求发送以后，我们不需要一直在等待服务端的响应，而是指定一个回调函数告诉XMLHttpRequest如何处理服务器返回的结果，剩下我们就可以继续处理其它任务。下面的代码不仅展示了AJAX如何异步调用，而且也印证了关注点分离，任何与XMLHttpRequest无关渲染工作都开放给用户来指定。

```javascript
	xmlHttpRequest.onreadystatechange = function()
	{
		if (xmlHttpRequest.readyState == 4 
			&& xmlHttpRequest.status == 200)
		{
			document.getElementById("elementId")
				.innerHTML = xmlHttpRequest.responseText;
		}
	}
	xmlHttpRequest.open("GET", url, true);
	xmlHttpRequest.send();
```

> ## 观察者模式(Observer Pattern)

回调同时也是一个简单的观察者模式，从第二个例子可以发现XMLHttpRequest就是一个Subject，而回调函数就是Observer，onreadystatechange事件一旦触发，XMLHttpRequest就会调用其注册的Observer方法。如此重要的特性，酷爱给开发者创造语法糖的C#自然也不例外，于是C#提供了一个新的关键字delegate，去定义一种新的类型来引用某种类型的函数。

# 2.不仅仅是函数引用

如果你把委托想象成函数或函数指针，那你就低估了委托的功能。 C#是面向对象的语言，所以委托的本质其实是一个对象(delegate是类型安全的)，比如下面代码
```C#
public delegate void Output(string name);
//delegate使用
public static void NotifyMessage(string name)
{
	System.Console.WriteLine(name);
}
Output output = NotifyMessage;
//delegate调用
output("hello delegate");
```
_Intermediate Language_

```IL
.class auto ansi sealed nested public Output
       extends [mscorlib]System.MulticastDelegate
{
	.method public hidebysig newslot virtual 
	        instance class [mscorlib]System.IAsyncResult 
	        BeginInvoke(string name,
	                    class [mscorlib]System.AsyncCallback callback,
	                    object 'object') runtime managed
	{
	}
	.method public hidebysig newslot virtual 
	        instance void  EndInvoke(class [mscorlib]System.IAsyncResult result) runtime managed
	{
	}
	.method public hidebysig newslot virtual 
	        instance void  Invoke(string name) runtime managed
	{
	}
}
```
简单的一行代码，翻译成中间语言变成了一个类。这个类继承了MulticastDelegate并且是sealed class不能再被其它对类型继承。这里面有三个函数，其中BeginInvoke，EndInvoke是为了实现异步回调，而Invoke则是同步调用。当然delegate的基类MulticastDelegate也提供了大量高级方法，下面我们就看一个例子

```C#
using System;

delegate void SayDel(string s);

class TestClass
{
    static void Hello(string s)
    {
        System.Console.WriteLine("  Hello, {0}!", s);
    }

    static void Hi(string s)
    {
        System.Console.WriteLine("  Hi, {0}!", s);
    }

    static void Main()
    {
        // 委托声明.
        SayDel hiDel, helloDel, multiDel, minusDel;

        hiDel = Hi;
        helloDel = Hello;

        // 合并hiDel, byeDel 
        multiDel = hiDel + helloDel;

        // 移出hiDel委托
        minusDel = multiDel - hiDel;

        hiDel("A"); // Hi, A!
        helloDel("B"); // Hello, B!
        multiDel("C"); // Hi, C! Hello C!
        minusDel("D"); // Hello D!
    }
}
```
上面的例子我们发现委托可以相加或相减。相加后的委托在被调用时会触发调用所有相加的委托，而相减则相反。而这些操作都是由Delegate(MulticastDelegate的基类)中的Combine和Remove来完成(C#重载了+=，-=，+，-)，它们共同维护一个Delegate的数组。当委托被调用时，编译器会生成遍历这个数组并依次调用Delegate对象的Invoke方法的代码。当然这样做会有些问题我们无法控制，比如上面multiDel如有有返回值，那么调用multiDel("C")只会返回最后一个函数的返回值或则第一个函数有异常抛出，那么整个遍历就会终止。如果我们正真的很感兴趣返回值或异常。那么我们如何去控制呢？其实很简单，在基类MulticastDelegate中提供了一个接口GetInvocationList，你可以通过这个接口获得一个Delegate数组，然后自己遍历这个数组并处理其中的异常和返回值。

# 3.匿名函数和lamdab表达式
有时为了方便，我们希望定义一些函数临时的处理一些逻辑，而这些行为既不是对象行为的抽象，也不会被多次重复调用，这时我们可以用delegate去定义一个匿名函数，来减少单独定义函数的开销。比如下面代码

```C#
delegate int Add(int a, int b);

//Part1 匿名函数定义
Add Sum;
{
	int sum = 10;
	Sum = delegate(int a, int b){
		return sum + a + b;
	}
}
Sum(1, 2);

//Part2 具名函数委托
class Test{
	private int _sum = 0;

	public Test(int sum){
		_sum = sum;
	}

	public int Add(int a, int b){
		return a + b;
	}
}
int sum = 10;
Test test = new Test(sum);
Add Sum = test.Add;

Sum(1, 2);
```
代码Part1是匿名函数，你会发现匿名函数可以访问其作用域以外的变量sum，甚至sum出了作用域，匿名函数内的sum变量依然有效。其实这就是一个典型的闭包概念,匿名函数可以捕获作用域外面的变量，与本地变量不同，捕获的变量会扩展其生存期，直到引用该匿名方法委托被垃圾回收。 Part2就是模拟了C#底层的实现方式，是不是觉得Part1的代码更加简洁方便。

当然C#3.0以后，尤其是LINQ的广泛使用，用delegate创建匿名函数都显得有些繁琐，于是lambda表达式开始取代大部分匿名函数实现。比如上面的匿名函数就可以像下面这种实现
```C#
Sum = (a, b) =>{ return sum + a + b;}
```
这里lambda表达式并没有指明参数类型，而是直接类利用型推导。是不是比匿名函数更简洁。lambda表达式的出现使得C#语言呈现出更多函数式编程的影子。

当然不是任何函数我们都得用匿名函数和lambda表达式让代码显得简洁紧凑，这样反而会让代码不易调试，而且C#是面向对象语言，我们需要符合面向对象设计原则，保持代码的一致性，从而增加可维护性，所有新技术的的应用不是为了盲目追赶潮流，而是如何让代码更加清晰和可维护，且不可本末倒置。

# 4.事件(event)特殊的委托
很多场景我们需要在一个对象发生变化时，通知另外一些相关对象，并让这些对象根据变化做出相应的反应。这就是典型的观察者(Observer)模式或叫订阅/发布(subscribe/publish)模式。尤其在Windows平台很多设计都是基于事件消息机制，所以C#便提供了一个关键字event，让事件这个概念在C#中变得更加重要。下面就让我们看下事件是如何定义和使用的

```C#
class Subject{
     public event EventHandler<TestEventArgs> TestEvent;
     public void NotifyMessage(){
           if(TestEvent != null){
                 TestEvent(this, null);
           }
     }
}

class Observer1{
     public Observer1(Subject sub){
             sub.TestEvent += DoAction1;
     }
     public void DoAction1(object sender, TestEventArgs e){
            System.Console.WriteLine(“Observer1”);
     }
}

class Observer2{
     public Observer2(Subject sub){
            sub.TestEvent += DoAction2;
     }
     public void DoAction2(object sender, TestEventArgs e){
     System.Console.WriteLine(“Observer2”);
     }
}

class Program{
      public static void Main(){
            Subject sub = new Subject();
            Observer ob = new Observer(sub);
            sub.NotifyMessage();
      }
}
```
这里的TestEvent就是一个加了event限定的委托方法。 到了这里你是不是有点疑惑了？为什么要加event这个关键字，这不是画蛇添足吗，这里即使不需要event同样也可以达到预期的效果，那么为什么要加event呢？我们先看一段代码：

```#
Subject sub = new Subject();
Observer ob = new Observer(sub);
sub.TestEvent(sub, null);
```
上面第三行代码编译时会发生错误。之所以错误是因为event封装了事件的注册和调用，它变成Subject的“私有”成员。这就是面向对象设计的三大特征之一(封装)，我们应该尽可能多的封装外部不必要访问的内容和状态，以免外部意外改变和错误的调用。
同样委托这里就不会有任何限制。

# 5.后记
委托你可以把它当作C#提供给开发者的语法糖，当然如果你只停留在使用他的便捷，而不知道其所以然，那么傻瓜的方式会让你变得更傻瓜。相反你会站在巨人的肩膀上，将他人的思想变成自己的智慧。

