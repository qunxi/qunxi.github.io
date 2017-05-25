---
author: qunxi
create: 2017-02-28 18:43+08:00
update: 2017-03-04 18:43+08:00
layout: page
title: "设计模式-Builder模式"
description: ""
comments : true
categories:
tags:
- 设计
- 模式
---
# 前言

前面两篇文章分别介绍了[Factory模式](https://qunxi.github.io/2017/02/18/design-pattern-factory.html)和[Singleton模式](https://qunxi.github.io/2017/02/24/design-pattern-singleton.html)，今天我们再介绍另一种比较常用的构建型模式-Builder模式，很多开发人员很容易将它和[Factory模式](https://qunxi.github.io/2017/02/18/design-pattern-factory.html)混淆,尤其是抽象工厂，但是无论从使用场景或实现细节还是有明显区别的，今天我们就聊聊Builder模式，在阅读本文前，我建议您先阅读[Factory模式](https://qunxi.github.io/2017/02/18/design-pattern-factory.html)。
<!--more-->

# 1. Factory模式的重构

本文我们将继续引入之前的例子-汽车检测车间。为了便于描述Abstract Factory和Builder模式的区别。我们先回顾下Abstract Factory的代码。

```
interface IFactory{
     Car Create();
     Engine CreateEngine();
     Wheel CreateWheel();
}

class HondaFactory : IFactory{
     public Car Create(){
         Car car = new Honda();
         car.SetupEngine(CreateEngine());
         car.SetupWheels(CreateWheel(), 
                CreateWheel(), 
                CreateWheel(), 
                CreateWheel());
         return car;
     }

     public Engine CreateEngine(){
         return new HondaEngine();
     }

     public Wheel CreateWheel(){
         return new HondaWheel()
     }
}

class BenzFactory : IFactory{
     public Car Create(){
         Car car = new Benz();
         car.SetupEngine(CreateEngine());
         car.SetupWheels(CreateWheel(), 
                CreateWheel(), 
                CreateWheel(), 
                CreateWheel());
         return car;
     }
     
     public Engine CreateEngine(){
         return new BenzEngine();
     }

     public Wheel CreateWheel(){
         return new BenzWheel();
     }
}
```
如果你是一个代码"DRY-(Don't Repeat Yourself)"的拥护者也许你已经无法容忍上面的代码。在上面的代码中，两个工厂子类的`Create`方法，有着相似的组装流程。我们是否可以抽象提取这个相同的构建行为，从而可以在不同地方复用。如果你有类似的想法，那么你已经自行领悟到了Builder模式的真谛。下面我们使用Builder模式来重构上面的代码。

# 2. Builder模式

在开始重构代码之前我们先看看该模式的定义：

> **将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示** 

Builder模式更强调的是**构建过程**而不是构建的对象，这也是它和Abstract Factory的最主要的不同。下面是我们对上面代码的重构:

```
class Builder{
     protected Builder(){}

     public virtual void BuildFrame(){}
     public virtual void BuildEngine(){}
     public virtual void BuildWheels(){}
}

class BikeBuilder : Builder{
     private Bike bike;

     public void BuildFrame(){
        bike = new Forever();
     }

     public void BuildWheels(){
         Wheel w1 = new ForeverWheel() ;
         Wheel w2 = new ForeverWheel() ;
         bike.SetupWheels(w1, w2);
     }

     public Bike GetBike(){
         return bike;
     }
}

class BenzBuilder : Builder{
     private Car car;

     public void BuildFrame(){
        car = new Benz();
     }

     public BuildEngine(){
        car.setupEngine(new BenzEngine());
     }

     public void BuildWheels(){
          Wheel w1 = new BenzWheel();
          Wheel w2 = w1.clone();
          Wheel w3 = w1.clone();
          Wheel w4 = w1.clone(); 
          car.setupWheels(w1, w2, w3, w4);
     }

     public GetCar(){
         return car;
     }
}

class AssmablyLine{
    public void Assmable(Builder builder){
        builder.BuilderCar();
        builder.BuilderEngine();
        builder.BuilderWhheel();
    }
}
```
客户代码的使用

```
class ExamWorkshop{
   public bool Test(){
       AssmablyLine assmablyLine = new AssmablyLine();
       BenzBuilder builder = new BenzBuilder();
       assmablyLine.Assmable(builder)
       Car car = builder.GetCar();

       car.Run();
       return Verify(car);   
   }
}

```
![builder pattern](\post-images\2017_3_3_builder_pattern.png)

上面的代码我们做了一下几点调整：

1. **提取构建过程**：我们将组装过程抽象成装配流水线(`AssmablyLine`)，这样不同产品只要组装过程是一致的就可以复用这段代码，我们不再需要像工厂类一样重复实现一样的构建代码。这里的`AssmablyLine`就是Builder模式中的`Director`对象。

2. **“空”基类**：这里我们能将Builder变成一个什么都不做的“空”类，它既不是接口也不是一个抽象类。这样做的目的其实是为了让子类拥有默认构造行为，同时可以根据需要来重写不同部件的构建。比如例子中的`ForeverBuilder`因为自行车不需要构建引擎，所以我们可以直接使用基类的默认空函数，从而保证构建流程的一致性。当然为了让基类`Builder`拥有抽象接口的意义，我们将它的构造函数定义为`protected`，从而避免用户可以直接实例化Builder基类。

3. **构建非同类产品**：最后一点其实我们已经发现，我们可以通过Builder模式构建不同类型的产品(`Bike`和`Car`)，只要它们的构建过程是相同的。Builder模式不仅封装具体对象的构建，而且将构建的过程和构建对象进行了分离，这也是关注点分离的体现。

4. **Prototype模式**：在构建`BenzWheel`时，我们使用了Prototype模式（原型模式），如果一个对象构造的过程是复杂的，那么如果在我们已经拥有一个实例对象时，我们可以直接`clone`出一个新的对象，这样比重新构造要简单的多，虽然这里的`BenzWheel`构建并不复杂，这里的主要目的还是为了简单的介绍下原型模式的使用场景。 

# 后记

到目前为止我们已经将**构建型模式**介绍的差不多了，它们的主要目的就是为了封装构建对象以及构建过程，使客户代码完全独立于具体对象以及构建过程。当然我们并不是所有的对象构造我们都需要使用这些模式，有时直接`new`一个简单对象，反而比套用模式使代码变的更复杂来的方便。

