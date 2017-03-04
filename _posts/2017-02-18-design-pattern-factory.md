---
author: qunxi
create: 2017-02-17 18:43+08:00
update: 2017-02-18 18:43+08:00
layout: page
title: "设计模式-Factory模式"
description: ""
comments : true
categories:
tags:
- 设计
- 模式
---
# 前言

在[上篇文章](https://qunxi.github.io/2017/01/30/object-oriented-design.html)我们介绍了代码设计的理论概念，它是我们做出好的设计的指导原则-**"道"**。而在现实开发过程中我们通过经验总结出许多处理相同场景的模式来满足这些指导原则，这就是我们常说的设计模式-**"术"**。这篇文章我们将从最简单的工厂模式开始。
<!--more-->

# 1. 从`new`到Factory

在面向对象语言中我们一般通过`new`来构建一个对象，但是`new`的出现就意味着强依赖的发生。为了对具体构造对象进行解耦，我们需要引入新的一层，来隐藏构建的过程，这就是创建型模式存在的意义。而`Factory`模式是这种模式的典型代表，为了便于对`Factory`模式的理解，我们先引入一个场景：假设有一个汽车检测车间（ExamWorkshop），专门负责抽样检测轿车的功能、质量、性能等问题。为了实现上面的功能我们定义了以下类型。

```
interface Car{
    void SetupEngine(Engine engine);
    void SetupWheels(Wheel w1, Wheel w2, 
                     Wheel w3, Wheel w4);
    void Run(); 
    ...
}

class Honda : Car{
    ...
}

// ExamWorkshop负责对Car进行检查
class ExamWorkshop{
   public bool TestCar(){
         Car car = new Honda();

         Engine engine = new Engine();
         car.SetupEngine(engine);

         Wheel wheel1 = new Wheel();
         Wheel wheel2 = new Wheel();
         Wheel wheel3 = new Wheel();
         Wheel wheel4 = new Wheel();
         car.SetupWheels(wheel1, wheel2, 
                         wheel3, wheel4);
         car.Run();
         return Verify(car); 
    }

    private bool Verify(Car car){
        //检测功能、安全，性能报告
        .....
    }
}
```
# 2. 简单工厂（Static Factory Method）

从功能实现上看上面的代码好像没什么问题，但是无论从功能职责上的划分还是日后的扩展都留下了一定的隐患。如果有一天公司需要支持检测更多的轿车品牌（Benz、BMW、Toyota），那么修改`ExamWorkshop`的可能性就非常大。从单一职责（Single Responsibility）考虑，作为检测车间是不应该负责汽车构建的，构建的职责应该由汽车工厂负责。另外作为检测车间是不应该强依赖于具体汽车品牌（Honda），否则一旦汽车品牌更新，那么检测车间也就发生变化（比如在`class ExamWorkshop`实现多套类似`TestCar`的函数）。针对上面提到的两点我们需要对代码做一些重构。

```
class CarFactory
{
    public enum Brand{
        Honda,
        Benz,
        BMW,
        Toyota 
    }

    public static Car Create(Brand brand){
        switch(brand){
            case Honda:{
                Car car = new Honda();
                return AssmableCar(car);
            }
            case Benz:{
                Car car = new Benz();
                return AssmableCar(car);
            }
        }
    }

    private void AssmableCar(Car car){
        Engine engine = new Engine();
        car.SetupEngine(engine);

        Wheel wheel1 = new Wheel();
        Wheel wheel2 = new Wheel();
        Wheel wheel3 = new Wheel();
        Wheel wheel4 = new Wheel();
        car.SetupWheels(wheel1, wheel2, wheel3, wheel4);
    }          
}

class ExamWorkshop{
   public bool TestCar(CarFactory.Brand brand){
         Car car = CarFactory.Create(brand);
         car.Run();
         return Verify(car);   
   }
}
```
![static factory method](/post-images/2017_2_18_static_factory_method.png)

上面的代码做了两点重构，首先我们将汽车构建的过程移到了`CarFactory`这个**简单工厂**里面，其次`ExamWorkshop`不再依赖具体的汽车品牌对象比如Honda，而是抽象接口`Car`，当然我们还将可复用的组装流程提取成一个私有函数`AssmableCar`。**简单工厂**又叫静做*静态工厂方法*，它最主要的部分就是那个静态`Create`函数，它负责封装所有的构建过程，这样`ExamWorkshop` 对汽车的构建过程将一无所知，而且事实上它也不应该知道具体的实现细节。另外通过**简单工厂**返回的抽象接口，我们将`ExamWorkshop`和具体的汽车品牌进行了解耦。

# 3. 工厂方法（Factory Method）

通过**简单工厂**虽然我们解决了具体品牌汽车与`ExamWorkshop`的依赖，但是我们又引入了另一个依赖问题，`ExamWorkshop`和`CarFactory`发生了紧耦合，而且一旦我们增删了具体汽车品牌，`ExamWorkshop`也会间接的受到影响。如果汽车构建过程和汽车品牌相对稳定这样设计并没有什么太大的问题，否则**简单工厂**并没有比一开始的代码带来更多的改进（除了业务职责划分更具针对性）。**简单工厂**本质上其实是违背*开闭原则（Open Close Principle）*的，为了解耦`ExamWorkshop`和`CarFactory`的依赖，我们可以通过**工厂方法**来提取一个抽象类，让`ExamWorkshop`只依赖于抽象而不依赖于具体的工厂实现。下面我们进一步的重构代码

```
public abstract CarFactory{
    
    public abstract Car Create();

    protected Car AssmableCar(Car car){
        Engine engine = new Engine();
        car.SetupEngine(engine);

        Wheel wheel1 = new Wheel();
        Wheel wheel2 = new Wheel();
        Wheel wheel3 = new Wheel();
        Wheel wheel4 = new Wheel();
        car.SetupWheels(wheel1, wheel2, wheel3, wheel4);
        return car;
    }
} 

public class BenzFactory : CarFactory{

    public override Car Create(){
       Car car = new Benz();
       return AssmableCar(car); 
    }
}

public class HondaFactory : CarFactory{

    public override Car Create(){
       Car car = new Honda();
       return AssmableCar(car); 
    }
}

class ExamWorkshop{
   public bool TestCar(CarFactory factory){
         Car car = factory.Create();
         car.Run();
         return Verify(car);   
   }
}
```
![factory method](/post-images/2017_2_18_factory_method.png)

上面我们将`CarFactory`提成抽象类，并为不同的汽车品牌定义相应的具体工厂，比如`BenzFactory`、`HondaFactory`等。`ExamWorkshop`已经不再依赖于具体的工厂而是抽象类`CarFactory`，这样`ExamWorkshop`将不再受汽车品牌以及工厂的变化而变化，如果需要支持新的品牌，我们只需要扩展一个具体品牌的汽车和具体的工厂方法类，而不需要修改客户类`ExamWorkshop`，这正好弥补了**简单工厂**破坏*开闭原则*的缺点。那这是不是一个很完美的解决方案呢？当然不是， 你会发现随着支持的汽车品牌越来越多，我们需要为每一个新的品牌添加新的工厂，这无形中给自己增加了重复性工作，并是代码急剧膨胀，而且`BenzFactory`和`HondaFactory`的内部实现几乎是很相似的，除了具体的汽车类型不同。为了解决这样重复创建新工厂的弊端，我们可以通过模板来重构上面的代码。

```
class ConceretCarFactory<T> : CarFactory where T: Car, new(){
    public override Car Create(){
        Car car = new T();
        return AssmableCar(car);
    }
}
```
现在我们只管添加或删除汽车品牌，而不需要提供相对应的工厂类。**工厂方法**和**简单工厂**相比，它的可扩展性更高

# 4. 抽象工厂（Abstract Factory）

上面的代码假设所有的汽车使用都是一样的发动机或轮胎，但是现实中不同汽车使用的发动机、轮胎等部件都是不一样的。所以构建的事物由一系列组件组成时，我们需要使用**抽象工厂**来完成构建的过程。**抽象工厂**是一系列**工厂方法**的集合，比如下面的代码，我们在具体的`HondaFactory`以及`BenzFactory`中实现了不同`Engine`和`Wheel`的构造方法。

```
interface Engine{

}

class BenzEngine  : Engine{

}

class HondaEngine : Engine{

}

interface Wheel{

}

class BenzWheel : Wheel{

}

class HondaWheel : Wheel{

}

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
         return new HondaWheel();
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
![abstract factory](/post-images/2017_2_18_abstract_factory.png)

**抽象工厂**抽象出了一系列组件及事物的构造方法，但是它也有自己的局限性，如果要添加或修改部分抽象工厂的构建方法，那么就需要重新抽象基类，这会影响已经存在的具体工厂，同时抽象基类变化，那么也会影响到它的客户代码。

工厂模式主要的目的是为了**封装构建过程，并隔离客户代码对具体被创建对象的依赖**。但是每一种工厂模式都有自己的优势和缺点，我们必须根据具体的场景来选择适合自己的模式。

# 后记

工厂模式也有其自身的问题，比如引入了抽象工厂的依赖，虽然这不是太大的问题，但是从`ExamWorkshop`的角度来说它应该只关心`Car`，而不应该知道`Factory`。如何解决这些问题，这里我们暂时打个问号，如果你很感兴趣可以去了解下**依赖注入**和**IoC容器**，这些我将在以后的文章中继续讨论。最后我想说的就是模式是在实践中产生的，它并不是固定不变的，它是经验的总结，另外模式是对某些场景的总结，它的选择需要很多的权衡，我们不能为了模式而模式。

## （转载本站文章请注明[作者和出处](https://qunxi.github.io/)，请勿用于任何商业用途）



 