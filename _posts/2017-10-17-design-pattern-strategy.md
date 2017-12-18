---
author: qunxi
create: 2017-08-27 18:43+08:00
update: 2017-08-27 18:43+08:00
layout: page
title: "设计模式-Strategy模式"
description: ""
comments : true
categories:
tags:
- 设计
- 模式
---
# 前言

曾经看到过这样一句话“`if-else/switch`是反模式(anti-pattern)的”。这句话非常有意思，但是我并不是很认同，因为设计本身就不是绝对的，如果一句话是绝对的那么它本身就是反设计的。当然这句话并不是完全没有指导意义。比如Strategy、State等行为模式就是在解决`if-else/switch`在一些场景中不完美的实现，下面我们一起来学习下Strategy模式:
<!--more-->

# 1. 无处不在的条件判断(if-else/switch)

程序中所有算法都是逻辑的表达，我们可以利用`if-else/switch`来构建复杂的条件判断，比如不同状态(State)、命令(Command)或策略(Strategy)的逻辑处理。下面我们来看看条件判断在例子中的实现，假设我们要实现一个节假日商品促销程序，程序可以根据不同的节假日给出商品的不同促销价格。下面是程序代码的实现：

```C#
public enum Holiday{
    SpringFestival,
    MayDay,
    NationalDay
};

public class Product
{
    public double Price {get; set;}

    public Product(){
    }

    public double GetPromotionPrice(Holiday holiday)
    {
        switch(holiday) {
            case SpringFestival:
                return Price * 80%;
            case MayDay:
                return Price * 85%;
            case NationalDay:
                return Price * 90%;
            default:
                return Price;
        }
    }
}
```

以上代码非常简单，整个逻辑主要集中在`GetPromotionPrice`这个方法中，我们通过`switch`的条件判断来执行不同节假日的促销策略。一般这样实现已经非常清晰，但是如果商场促销活动非常多，而且促销的策略变化也非常频繁，那么这样实现就会非常的脆弱，因为任意修改或添加新的促销方案我们都必须修改`Product`对象(违背了**Open Close Priciple**)，而理论上这些变化是不应该影响到`Product`对象。归其原因就是`Product`对象中依赖(包含)了促销策略这个变量(违背**Single Responesiblity Priciple**)。为了隔离出这个变化点，下面我们通过一些小小的重构来解决上述问题，从而实现注点分离(**Seperation of Concern**)。

# 2. Strategy模式 - 条件判断终结者

为了剥离出`Product`对促销策略的依赖。我们抽象出一个`PromotionStrategy`抽象类，同时我们将之前条件策略封装成派生于`PromotionStrategy`的具体子类。一下便是抽象出来的策略对象：

```C#
public abstract class PromotionStrategy{
    public abstract double GetPrice(double price);
}

public class SpringFestivalStrategy : PromotionStrategy{
    public override double GetPrice(double price){
        return price * 80%;
    }
}

public class DoubleElevenStrategy : PromotionStrategy{
    public override double GetPrice(double price){
        return price * 60%;
    }
}

public class MayDayStrategy : PromotionStrategy{
    public override double GetPrice(double price){
        return price * 85%;
    }
}

public class NationalDayStrategy : PromotionStrategy{
    public override double GetPrice(double price){
        return price * 90%;
    }
}
```

提取完促销策略对象，我们再来看看客户代码-`Product`对象，通过构造依赖注入(**Dependency Inject**)我们可以隔离`Product`对象对具体策略类的依赖。这里`Product`对象只会依赖于抽象类`PromotionStrategy`.

```C#
class Product
{
    private PromotionStrategy _strategy;

    public Product(PromotionStrategy strategy){
        _strategey = strategy;
    }

    public double Price {get; set;}

    public double GetPromotionPrice(){
        return _strategy.GetPrice(this.Price);
    }
}
```

好了现在你会发现`Product`的代码变得更加简单独立，促销策略的变化已经不会影响到`Product`，与此同时这些策略、算法又可以被复用在其他代码中。当然这里也存在一些缺点，比如我们在隔离变化点的同时，增加了对象的数目,在一定程度增加了维护成本。到底孰优孰劣，我想判断的标准是你在代码维护过程中，你的痛点到底是什么？是牵一发而动全身的变化扩散，还是已经影响到你开发效率的对象管理。

```C#
class Product<TStrategy>
{
    private TStrategy _strategy;

    public double Price {get; set;}

    public double GetPromotionPrice()
    {
        return _strategy.GetPrice(this.Price);
    }
}
```
我们也可以通过模板的静态绑定来减少继承关系以及对象数量，比如上面的实现我们就不再需要抽象类`PromotionStrategy`，因为是编译期绑定，所以可以在一定程度提高效率。当然这样做的是牺牲动态调整策略换来的。

# 3. 举一反三 - 行为对象的封装

以上两种实现就是我们所介绍的Strategy模式，下面我们来看看Strategy模式的定义:

> 定义一系列的算法，把它们一个个封装起来，并且使它们可相互替换。本模式使得算法可独立于使用它的客户而变化。

![strategy pattern](/post-images/2017_10_20_strategy_pattern.PNG)

Strategy模式将**易变性(volatility)**的算法提取封装成独立于客户代码的对象，从而使客户代码的职责更加清晰单一，并且提高了算法的复用性。

在行为模式中有很多类似以上目的模式，比如Comand模式：它将多样性的请求封装成对象，并独立于客户代码；State模式：它将客户代码的内部状态，提取封装成状态对象，使得客户代码完全不用关心内部状态发生变化后所对应行为。这些都是在隔离变化，提高复用。具体这些模式的介绍，我们在其他篇幅介绍。

# 后记

在现实开发中我们应该像上面的例子一样，先使用`if/switch`来实现功能，直到有一天你发现，一些算法你希望得到重用又或者某一天你发现业务的扩展和变化使得算法也不停变化时，这个时候也许我们应该开始考虑使用Strategy模式。如果算法和客户代码都是稳定的，那么我们就没有必要再画蛇添足。
