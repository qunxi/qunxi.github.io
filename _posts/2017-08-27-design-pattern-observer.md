---
author: qunxi
create: 2017-08-27 18:43+08:00
update: 2017-08-27 18:43+08:00
layout: page
title: "设计模式-Observer模式"
description: ""
comments : true
categories:
tags:
- 设计
- 模式
---
# 前言

如果要我选出使用频率最高的三个设计模式，我会选择[Factory模式](https://qunxi.github.io/2017/02/18/design-pattern-factory.html)、Observer模式以及其它模式（其它模式会因为项目而有所不同）。所以这篇文章我们一起来了解下Observer模式。

<!--more-->

# 1. 如何通信

[代码设计](https://qunxi.github.io/2017/01/30/object-oriented-design.html)就是一个管理复杂度的过程，我们擅长使用**分治法**将一个复杂的问题分解成多个相对容易的问题，然后各个击破，在软件设计中我们也叫做解耦。解耦我们通常遵循**高内聚低耦合**原理。比如在开发一个用户注册系统，如果用户注册成功，我们会给用户的邮箱发一份邮件。从业务领域分析，很明显我们需要拆分两个相对独立的领域对象`User`、`EMail`。

```
class User
{
    private Email _email = new Email();

    public bool SignIn(string username, string password, string mail)
    {
            ····
        _email.Send(mail, "congratulations to u");
    }
}

class EMail
{
    public bool Send(string address, string content){

    }
}
```

上面的代码存在以下几个问题：

1. 缺乏扩展的灵活性：比如用户如果使用手机号或微信号注册，那么`User`对象必须得引入类似`TextMessager`或`WeChatMessager`之类的对象，来处理消息的通知。每添加或修改一个被通知对象就会影响`User`对象。

2. 实现的依赖倒置：设想如果我们需要通过用户界面通知用户注册成功。那么是不是`User`对象还要引入UI对象(`View`)的依赖呢？可以想象业务对象依赖上层UI是多么可怕的噩梦(难以单元测试不说，更重要的是业务逻辑不易复用，注：这里的`依赖倒置`并非Solid原则中的接口依赖倒置)

如何解决上述问题？相信学习了那么多的设模式的你应该已有一个初步的想法-提取抽象，让`User`对象不再依赖具体被通知对象。下面我直接利用Observer模式来重构上面的代码。

# 2. 消息的订阅与发布-Observer模式

在开始代码重构之前，我们还是先来看看Obeserver模式的定义。

 > 定义对象间的一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。

接下来让我们结合定义来理解下面的代码，这里有必要先阐述两个重要对象`Observer`和`Subject`

* Observer(观察者)：消息的订阅者，负责接收消息后，执行相应的更新操作。

* Subject(目标对象)：消息的发布者，负责维护与观察者的订阅关系。

```
abstract class Subject
{
    private List<Observer> _observers;

    public void Attach(Observer o){
        _observers.Add(o); 
    }

    public void Detach(Observer o){
        _observers.Remove(o);
    }
    
    public void Notify()
    {
        foreach(var o in _observers){
            o.Update();
        }
    }
}

class User : Subject{
   
    public bool SignIn(string username, string password, string mail)
    {
        ····
        // 注册成功
        Notify();
    } 
}

abstract class Observer{
    public abstract void Update();
}

class Email : Observer{
    public override void Update(){
        // 发送邮件
    }
}

class View : Observer{
    public override void Update(){
        // 更新用户界面
    }
}

class TextMessager :Observer{
    public override void Update(){
        // 发送短消息
    }
} 
```

上面的代码，我们提取了`Observer`作为`TextMessager`、`Email`和`View`的基类，它提供了一个*Update*接口，所有派生类可以根据具体要求实现相应的*Update*行为。`User`作为被观察对象继承`Subject`类，所有观察者(或订阅者)通过*Attach*或*Dettach*方法和`Subject`对象建立或取消订阅关系。

通过上面的例子我们发现Observer模式很好的解耦了观察者`Email`、`View`和被观察者`User`的直接依赖引用，尤其是一对多的情况下，因为观察者与目标对象的完全隔离，使得任何一方的修改或扩展变得非常灵活-这就是**OPC(Open Close Principal)**原则，同时又能及时的通知相关对象及时更新。

当然观察者模式也有明显的缺点:

1. 目标对象的维护成本很高。比如在一对多的应用场景时，目标对象需要维护大量的订阅关系，而且如果观察者数量巨大容易发生性能问题。

2. 观察者状态更新出错时很难定位问题。因为所有观察者的更新状态都是由目标对象触发的，尤其如果一个观察者有观察多个目标对象，那么问题的定位就更加困难。

下面是Observer模式的UML图：

![observer pattern](/post-images/2017_8_27_observer_pattern.png)

# 3. Observer模式的扩展

## 1. 推拉模型

Observer模式可以根据不同的应用场景分为**Push**模型和**Pull**模型。上面的例子就是**Push**模型的实现方式，观察者不需要知道具体的目标对象，它们的更新都是被动的接收目标对象的触发。这样的好处是观察者可以更高效的处理自己的问题，但是这里获得的高效是通过牺牲目标对象性能换来的。有些时候目标对象成为了系统瓶颈，所以在一些目标对象性能要求很高的应用场景时，我们应该如何优化Observer模式呢？这里就是我们需要介绍的另一种模型**Pull**模型。**Pull**模型是由观察者主动承担更新操作，它必须知道具体的目标对象，通过目标对象获取更新的内容。这里的目标对象依然会通知观察者，告诉观察者目标状态已经发生变化，但是具体变化的内容以及何时更新都由观察者来决定。下面是**Pull**模型的实现代码，这里其他条件不变，唯一变化的是`Observer`对象多定义了一个*Update*函数。

```
abstract class Observer{
    public abstract void Update();
    public abstract void Update(Subject subject);
}

class Email{
    ...
    public override void Update(Subject subject){
        var data = subject->GetUpdateContent();
        // TODO 更新数据
        ...
    }
}
```
这里是不是需要subject提供一个*GetUpdateContent*接口这个可以根据具体应用进行抽象。因为目标对象很难抽象一个可以供所有观察者都适用的接口，所以这个可以根据具体情况而定。在两中模型都适用的情况下我还是建议使用**Push**模型。

## 2. C#的事件委托

所有模式本质都是在解决语言自身特性的缺陷或复杂度。比如面向对象语言本身就缺乏直接表述事件通知行为，但是这种特性的应用场景又非常普遍，所以我们总结出了Observer模式。与之类似的还有Iterator模式、Command模式等。当然灵活的语言设计本身就是多范式的，比如C#虽然是面向对象语言，但是也包含了很多函数式编程的思想。因为Observer的普遍应用，C#直接将Observer模式提取成了语言特性，我们可以通过delegate和event快速的实现消息通知订阅，下面我们来看看如何使用C#的event实现上面的例子。（具体C#的委托和事件介绍参看[C#委托(Delegate)](https://qunxi.github.io/2016/10/16/csharp-delegate.html)）

````
class User{

    public delegate void EventHandle();
    public event EventHandle Handle;

    public Notify()
    {
        handle();
    }
}

class Email{
    public static void SendMail(){
        // 发送邮件
    }
}

class Text{
    public static void SendText(){
        // 发送短消息
    }
}

class Program(){
    public void static Main(){
        User user = new User();
        user.Handle += new User.EventHandle(Email.SendMail);
        user.Handle += new User.EventHandle(Text.SendText);
        user.Notify();
    }
}
````
**event**和上面Observer模式的实现大同小异，它只是将维护订阅关系的逻辑提取成event作为目标对象的成员，而Observer模式是抽象到了基类。这里使用delegate的优势是我们不需要为观察者定义一样的函数签名，只要函数的出参和入参是一致的就可以。

# 后记

Observer模式的应用非常广泛，形式也非常多样。从更微观的层面其实回调函数也是Observer模式的一种实现形式，比如javascript被抱怨的Callback Hell就可想回调函数使用的普遍性。当然从宏观上讲服务与服务之间的异步、离线处理同样也有Observer的影子。所以说模式永远没有固定的形式，只是一个理念的不同情况的应用。
