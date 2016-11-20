---
author: qunxi
create: 2016-11-20 18:43+08:00
update: 2016-11-20 18:43+08:00
layout: page
title: "单元测试-实战篇(Moq)"
description: ""
comments : true
categories:
tags:
- 工程
- 基础
---
# 前言

在[上篇文章](https://qunxi.github.io/2016/11/18/unit-testing-mstest.html)我们以二分查找为例介绍如何写单元测试。而在现实的项目中，很多被测代码逻辑更加复杂，甚至还要依赖
其它对象才能完成任务。如果要编写这样的单元测试我们就会引入很多依赖(比如网络通讯等)，从而使编写单元测试逻辑变得复杂，运行效率低下。有时测试失败，可能并不是被测代码的逻辑错误
而是它的依赖代码出了错。在[基础篇](https://qunxi.github.io/2016/11/05/unit-testing-basic.html)中我们介绍了Test Double，下面我们利用Moq框架来介绍如何使用Mock或Stub在单元测试中消除依赖， 
是测试更专注于被测代码的逻辑。
<!--more-->

# 0x0001-测试场景

在开始介绍如何使用Moq进行Mock做Test Double之前，我们需要假设一个场景，方便理解上达成一致。我们都有过买火车票的经历，我们一般会在售票中心购买火车票，售票中心会根据你的需要（目的地，票的类型，票
的张数）售出火车票。这里我们在`TicketCenter`类中提供了一个`BuyTickets`函数负责购买火车票，而这个函数就是我们今天要测的代码。我们知道在买火车票时，`TicketCenter`应该去远程数据库(`TicketRepository`)
查询是否有足够的余留火车票，如果有支付成功后，通知售票机(`TicketMachine`)打印车票，否则提醒用户。

以下是三个主要独立的部件，代码片段3是我们需要测试的对象，而且其它两个依赖都涉及复杂的IO交互，这里我们假设依赖部分的代码已经有单元测试覆盖。

1 负责远程服务端数据库查询，更新

```
public enum TicketType{
     SoftSeat,
     HardSeat,
     NoneSeat,
}

public interface ITicketRepository
{
    int getTicketsLeft(TicketType type, string from, string to);
    bool BuyTickets(TicketType type, int count, string from, string to);
}

public class TicketRepository : ITicketRepository
{
    // 实现
    int getTicketsLeft(TicketType type, string from, string to){
    }

    bool BuyTickets(TicketType type, int count, string from, string to){
    }
}
```

2 负责打印火车票

```
public class TicketInfo{

}

public interface ITicketMachine
{
    bool PrintTickets(List<TicketInfo> ticketInfo);
}

public class TicketMachine: ITicketMachine
{

    bool PrintTickets(List<TicketInfo> ticketInfo){
        // 打印车票
    }

}
```
3 被测代码

```
public sealed class TicketCenter
{
    private ITicketRepository _ticketRepository = new TicketRepository();

    private ITicketMachine _ticketMachine = new TicketMachine();

    public bool BuyTickets(TicketType ticketType, int ticketCount, string from, string to)
    {
        int remainTicket = _ticketRepository.getTicketsLeft(ticketType, from, to);
        
        if(remainTicket < ticketCount){
            return false;
        }

        if(_ticketRepository.BuyTickets(ticketType, ticketCount, from, to)){
            
            List<TicketInfo> ticketInfo = new List<TicketInfo>();
            ....
            return _ticketMachine.PrintTickets(ticketInfo);
        }
        else{
            return false;
        }

    }

}
```
现在我们需要对`TicketCenter`的`BuyTickets`函数进行单元测试，这里我们还是用MsTest进行单元测试

```
[TestClass]
class TicketCenterTest
{

    TicketCenter _center;

    [TestInitialize]
    public void Setup(){
        _center = new TicketCenter();
    }

    [TestMethod]
    public void BuyTickets_BuyTwoTicketsSuccessfully_ReturnTrue(){
        //Arrange
        TicketType type = TicketType.SoftSeat;
        int count = 2;
        string from = "Shanghai";
        string to = "Beijing";

        //Action
        var ret = _center.BuyTickets(type, count, from, to);

        //Assert
        Assert.IsTrue(ret);
    }

    [TestMethod]
    public void BuyTickets_BuyTicketsCountLargeThanRemains_ReturnFalse(){
        //Arrange
        TicketType type = TicketType.SoftSeat;
        int count = 112;
        string from = "Shanghai";
        string to = "Beijing";

        //Action
        var ret = _center.BuyTickets(type, count, from, to);

        //Assert
        Assert.IsFalse(ret);
    }

}

```
是不是很完美？可是现实中这样的单元测试真的可以运行吗？很明显要跑通这个测试你需要链接到火车站的票务数据库，同时你还要在你的测试机旁边
放一台打印车票的机器，每跑一次单元测试，我们就需要去票务数据库查询一次，同时还要打印出那些没有意义的车票。说到这里，你也许会反驳我，傻
啊，为什么不链接一个本地的有测试数据的假数据库？还有可以做一个打印车票的仿真器(emulator)。如果你想到这点说明你已经具备了Test Double
的概念了。但是即使这样，这些Dummy Object还是太重量级，我们是做单元测试，不是集成测试(Integration Test)，我们要求单元测试应该是尽可能
快的运行，因为运行的频率很高，我们每提交一次代码就要跑一次。

# 0x0010-Stub/Mock与依赖注入（Dependency injection）

上面提到了Dummy Object太重量级了，那么是不是可以用些简单的方法解决这个问题呢？这里我们可以用Stub/Mock对象来替代之前提到的Dummy Object。

```
class TicketRepositoryStub : ITicketRepository
{
    int getTicketsLeft(TicketType type, string from, string to)
    {
        return 20;
    }

    bool BuyTickets(ticketType type, int count, string from, string to)
    {
        return true;
    }
}

class TicketMachineMock: ITicketMachine
{
    bool PrintTickets(List<TicketInfo> ticketInfo)
    {
        return true;
    }
}

```
这里我们定义了两个Test Double：`TicketRepositoryStub，TicketMachineMock`。也许你注意到这两个类的后缀一个是Stub，一个是Mock，就像在[基础篇](https://qunxi.github.io/2016/11/05/unit-testing-basic.html)
里介绍的Stub模拟的是对象的状态（TicketRepositoryStub返回的是剩余车票的期望张数），而Mock是模拟的是一种行为（我们只关心购买车票成功后，是否调用了打印车票
这个行为），两个Test Double对象实现后我们需要将它们和具体的依赖类（TicketRepository，TicketMachine）替换了。现在问题来了，在原来的设计基础上好像很难直接将依赖对象替换。
其实这个时候你也许发现了一个设计上的问题`TicketRepository`，`TicketMachine`和被测类`TicketCenter`有强耦合，`TicketCenter`类依赖了具体的实现，如果哪天
我们换了打印车票的机器或者其他的票务数据库那我们也要同时修改`TicketCenter`的代码，这违背了依赖倒置(Dependency Inversion)原则。
这里我们需要介绍一种设计模式-[依赖注入(Dependency Injection)](https://en.wikipedia.org/wiki/Dependency_injection)
依赖注入一般有三种形式：构造函数注入、Set函数注入（C#可以用Property注入）、接口注入。这里我将利用前两种重构`TicketCenter`类。
如对接口注入感兴趣，可自行查看上面的链接。

```
public class TicketCenter
{
    private ITicketRepository _ticketRepository;

    private ITicketMachine _ticketMachine;

    //构造函数注入
    public TicketCenter(ITicketRepository ticketRepository)
    {
           _ticketRepository = ticketRepository;
    }

    //属性注入（Set函数注入）
    public ITicketMachine TicketMachine
    {
        set{
            _ticketMachine = value;
        }
        get{
            return _ticketMachine;
        }
    }

    public bool BuyTickets(TicketType ticketType, int ticketCount, string from, string to)
    {
        int remainTicket = _ticketRepository.getTicketsLeft(ticketType, from, to);
        
        if(remainTicket < ticketCount){
            return false;
        }

        if(_ticketRepository.BuyTickets(ticketType, ticketCount, from, to)){
            //构造票务信息
            List<TicketInfo> ticketInfo = new List<TicketInfo>();
            ....

            if(_ticketMachine != null){
                return _ticketMachine.PrintTickets(ticketInfo);
            }
        }
        return false;
    }
}
```
代码重构完成，我们需要修改我们的单元测试（这里在强调一点，TDD的好处：其实单元测试就是你所实现代码的用户，如果你写的单元测需要很多依赖，说明你实现
的类有很多的耦合，那么其它产品代码使用你的类时也会有很多依赖，所以TDD有助于你写出低耦合的代码）。

```
    [TestInitialize]
    public void Setup(){
        //这里我们是直接构造，其实我们也可以用IOC容器或工厂类构建依赖对象
        ITicketRepository ticketRepository = new TicketRepositoryStub();
        _center = new TicketCenter(ticketRepository);
    }

    [TestMethod]
    public void BuyTickets_BuyTwoTicketsSuccessfully_ReturnTrue(){
        //Arrange
        TicketType type = TicketType.SoftSeat;
        int count = 2;
        string from = "Shanghai";
        string to = "Beijing";
        
        _center.TicketMachine = new TicketMachineMock();
        
        //Action
        var ret = _center.BuyTickets(type, count, from, to);

        //Assert
        Assert.IsTrue(ret);
    }
```
现在单元测试可以完全摆脱网路和打印机了，而且运行一次单元测试非常的快。其实到这里我们的单元测试介绍应该基本结束了，但是到这里还不完美。如果你是个“懒惰”的程序员
那么你会抱怨之前写的`TicketRepositoryStub，TicketMachineMock`需要大量的维护工作。比如之前的模拟都很简单，它完全不关心我的目的地，座位类型，一律返回20。如果
一些返回值和参数相关，并有复杂的逻辑，那么我就需要构建许多Stub/Mock对象。虽然不是很复杂，但是影响工作效率。于是乎就有一些人开始实现一些Mock框架来
帮助我们实现将这些繁琐没有技术含量的事，下面我们就介绍一个C#的Mock框架(Moq)。


# 0x0011-Mock测试框架（Moq）

在介绍Moq之前我先声明一点，这里的Mock框架已经不是单一的Mock，它还包含了Stub。还有为什么要以Moq为例，主要是它结合了函数式编程风格和Linq的特点，让代码看上去非常
简洁。其实Moq官网有很好的[GuidLine](https://github.com/Moq/moq4/wiki/Quickstart)，这里我只是结合自己的例子做一个简单的介绍。

```
    [TestInitialize]
    public void Setup(){
        _ticketRepositoryMock = new Mock<ITicketRepository>();
        _ticketMachineMock = new Mock<ITicketMachine>();
        _center = new TicketCenter(_ticketRepositoryMock.Object);
    }

    [TestMethod]
    public void BuyTickets_BuyTwoTicketsSuccessfully_ReturnTrue(){
        //Arrange
        TicketType type = TicketType.SoftSeat;
        int count = 2;
        string from = "Shanghai";
        string to = "Beijing";
        
         //Mock TicketMachine
        _center.TicketMachine = _ticketMachineMock.Object;
        _ticketMachineMock.Setup(mock => mock.PrintTickets(It.IsAny<List<TicketInfo>>())).Returns(true);

         //Mock Repository
        _ticketRepositoryMock.Setup(mock => mock.getTicketsLeft(type, from, to)).Returns(3);
        _ticketRepositoryMock.Setup(mock => mock.BuyTickets(type, count, from, to)).Returns(true);
        
        //Action
        var ret = _center.BuyTickets(type, count, from, to);

        //Assert
        //验证状态
        Assert.IsTrue(ret);
        //验证行为
        _ticketMachineMock.Verify(mock => mock.PrintTickets(It.IsAny<List<TicketInfo>>()), Times.Once);
    }
    
    [TestMethod]
    public void BuyTickets_BuyTicketsCountLargeThanRemains_ReturnFalse(){
        //Arrange
        TicketType type = TicketType.SoftSeat;
        int count = 10;
        string from = "Shanghai";
        string to = "Beijing";

         //Mock Repository
        _ticketRepositoryMock.Setup(mock => mock.getTicketsLeft(type, from, to)).Returns(3);

        //Action
        var ret = _center.BuyTickets(type, count, from, to);

        //Assert
        //验证状态
        Assert.IsFalse(ret);
        //验证行为
        _ticketMachineMock.Verify(mock => mock.PrintTickets(It.IsAny<List<TicketInfo>>()), Times.Never);
        _ticketRepositoryMock.Verify(mock => mock.BuyTickets(type, count, from, to), Times.Never);
    }

```
上面例子中Moq利用new Mock\<T\>构建一个Mock封装对象，其中T就是需要Mock对象的接口。然后Mock封装对象利用Setup去模拟接口函数的返回值以及异常等，当然还可以在Mock函数执行后
调用回调函数。最后我们可以通过Mock封装对象的Verify函数去验证需要验证的行为。这里要注意的是，要想获得真正的Mock对象，必须通过Mock封装对象的Object属性获得。当然Moq不仅可以做上面所说的事，
它还可以模拟C#中的事件(Event)以及依赖对象的保护成员。其实我所有的描述都有些多余，因为函数式的风格本身就有声明式的作用（只要知道What To Do 而不需要知道How To Do）。

# 后记

Mock对象是面向对象多态的基本应用，如果在开始写代码时你就考虑了依赖对象的Mock，那么你已经给自己定义了两种使用场景：一个是测试环境，一个是项目环境，这样有助于你在设计代码时理清依赖，因为测试环境有很多依赖是不存在的，所以它可以明显的提醒你对依赖进行解耦。好的设计是可以适应不同环境，做到开发封闭原则(Open Close Principle:对修改封闭，对扩展开发)，单元测试也在设计的角度保证了代码质量。