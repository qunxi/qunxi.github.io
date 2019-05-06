---
author: qunxi
create: 2019-04-23 18:43+08:00
update: 2019-05-01 18:43+08:00
layout: page
title: "微服务架构 - 服务的通讯"
description: ""
comments : true
categories:
tags:
- 基础
---

# 前言

在[前面的文章](https://qunxi.github.io/2018/12/25/microservice-architecture.html)，我们介绍了从工程角度将系统拆分成多个自治服务的好处，但从用户角度，用户并不希望去了解系统背后每一具体的服务，他们更关心的是整个系统的呈现方式以及整个系统的集成度。虽然集成是一个系统级别的概念，但它依赖于服务间可被理解的通讯，只有这样服务才可以相互调用。这篇文章我们一起聊一聊微服务间的通讯。
<!--more-->

# 1.远程调用(Remote Procedure Invocation)

服务通讯必须基于一套可被相互理解的协议(我们有时也叫接口或者契约，语言也是协议的一种呈现方式)，所以定义一套标准的协议会给服务开发带来巨大的经济效益。比如我们可以基于TCP等底层网络协议利用socket编程，实现服务间的通讯。但是从工程角度，我们每编写一个服务，我们就必须了解底层的通讯机制，而且经验有限的开发人员很容易将网络通讯的逻辑与业务代码相耦合，这不仅增加代码维护的复杂度还降低了开发效率。通过封装隐藏底层网络的细节，不仅可以让服务在逻辑上独立于网络基础层，而且有助于开发人员将更多的精力放在在实现服务逻辑本身。其次保证协议相对服务的独立，有利于服务的异构实现(不同平台，不同编程语言的实现)，从而有利于服务的分布式架构与部署。我们也可以基于应用层协议，比如HTTP(应用层协议本身就是对底层协议的封装，我们完全不需要重复造轮子)实现服务的通讯。

服务的远程调用一般可以分为以下两种方式。

* RPC(Remote Procedure Call)：对于一个没有网络编程经验的程序员来说，他们擅长的是进程内的函数调用，一旦要考虑跨进程或跨网络，就会引入很多复杂性(网络的不稳定，并发，限流等等)。如何让服务开发人员在对底层网络一无所知的情况下，又可以充分利用他们熟悉的方式进行编程？于是有人提出了RPC概念：顾名思义，我们可以像调用进程内函数一样，调用远程服务提供的函数，至于远程函数背后的技术细节都被隐藏在具体的RPC框架内。比如CORBA、gRPC、WCF等框架都是这种实现方式，他们不仅封装了底层网络的实现，而且还支持多种编程语言，并且跨平台(是的WCF支持有限)。因为这些框架底层的实现是平台、语言无关，所以造就了这些框架实现的服务具有很好的异构性。RPC调用的好处是传统的开发人员不需要太多转换成本，通过代理模式，业务层的开发完全屏蔽了网络细节，通过调用有业务含义的服务接口完成跨网络的通讯。

* REST(Representational State Transfer)：REST是完全基于HTTP实现的，它们不像RPC强调函数的调用，而是将业务对象看作一种资源，通过HTTP的动词(verbs:PUT/POST/GET/DELETE)和URL对业务资源进行操作，比如我们可以通过POST去创建一个资源，通过GET动词去查询一个资源或者通过PUT对已经存在的资源进行修改。REST和RPC相比更加简洁，它不需要像gRPC, CORBA那样引入一套IDL(接口定义语言)，去生成不同语言的代理客户端，只要客户代码支持HTTP访问，用户就可以接收并根据需要解析数据。因为REST是完全基于HTTP的，所以它并不会因为一些安全策略问题导致服务无法访问(一般机构的防火墙策略都支持HTTP访问)。当然因为REST并没有强制的约束，所以不同的人也会有不同的理解与实现，比如很多人并没有将REST当作一种资源的操作，它依然受RPC的影响，将服务以及服务的函数暴露在URL中(比如https://api.xxx.com/CustomerService/getCustomer/)。当然这并没有对错，只是从[REST成熟度模型](https://martinfowler.com/articles/richardsonMaturityModel.html)看，这并没有体现REST的最大精髓。

不论是RPC还是REST，要完成网络通讯必须具备三个基本要素(我们通常叫作ABC)：地址(Address)，绑定的传输协议(Binding)和数据契约(Contact)。地址最为简单，服务间的通讯必须知道具体的物理或逻辑地址进行连接(URL、IP+端口等)；不同框架支持的传输协议有所不同，一般都会实现一些通用的协议，比如TCP、HTTP等；数据契约也是一些平台无关的标准，比如SOAP(XML)、JSON、Protocol Buffers等。但是作为一个合格的服务接口(API)，这些信息必须对客户是隐藏的。

# 2.API网关(API Gateway)

随着服务的增加，服务接口的使用和管理将成为一个很大的问题。首先作为一个外部的应用程序，很可能需要调用多个服务才能完成一个具体的功能，所以客户必须对每个服务提供的接口非常了解，这将大大增加客户的学习负担，同时暴露过多的内部细节；其次每个服务中的接口并不是都需要暴露给外部客户的，一些接口只是为内部系统提供服务，所以我们需要为每个接口提供不同的访问权限和安全策略，这种分散式的管理(接口容易被忽视)，会增加了开发的成本；最后外部应用对服务的直接依赖，会导致因为服务的升级或修改，导致客户应用不能正常工作，这些便是我们开发微服务必须考虑的问题。

API网关有点类似面向对象设计模式中的门面模式(Facade Pattern)，我们可以将一些非功能性需求(路由、服务发现、负载均衡、安全认证等)以及粗粒度的聚合API在这层实现。这样既不会影响或修改已经存在的微服务，而且可以通过一个或几个集中的入口完成API的管理与控制。![APIGateway](https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/multi-container-microservice-net-applications/media/image39.png)

* API管理：RPI虽然在形式上将远程调用抽象成了类似本地的函数调用，但是我们依然要时刻提醒自己，每一次的远程函数调用都是非常大的消耗，我们应该尽可减少服务间的来回调用。比如一个具体功能需要调用多个服务的API才能完成，那么我们在设计外部API时，我就应该考虑是否应该提供一个面向业务的粗粒度接口来减少网络间的多次调用？另外外部应用只知道API网关提供的接口，其内部的调用关系对用户来说是黑盒，因为隔离了内部服务和外部服务的依赖，所以每个具体服务的改变都不会影响到外部应用。
* 服务发现与路由：我们知道服务发生通讯前必须根据网络地址建立连接。但是网络地址是一个变量，服务重启或者部署到不同的地方，网络地址都有可能发生变化。如果这些都由下游服务来负责管理，那么可以想象，一旦上游服务地址发生变化，下游应用将会无法工作。API网关的另一个好处就是，我们可以将API网关实现成一个反向代理，让所有服务在启动时将自己的地址注册到API网关，这样每一个服务在访问目标服务时，首先会通过API网管查询目标服务的位置，然后进行访问。这就是服务发现和路由的基本原理。
* 服务访问预处理：因为外部服务首先访问的是API网管，所以我们可以在API网关处理一些访问具体服务的预处理，比如权限的认证、API版本访问控制、负载均衡、服务断熔等。这些都属于cross-cutting的关注点，把它实现在API网关这层，正好实现了关注点的分离。

虽然API网关有很多好处，但是缺点也一样存在，因为API网关在设计上多加了一层，这不仅增加了开发成本，而且会带来性能的损耗，其次API网关很容易使系统架构趋向中心化，中心化的缺点就是瓶颈效应，一旦API网关宕机，没有灾备等措施，那么整个系统很可能瘫痪。

# 3.消息(Messaging)

远程函数调用的好处就是简单明了，但是它的缺点就是客户代码与服务接口存在直接的调用依赖，客户代码必须主动调用服务接口，服务接口一旦改变，客户代码也必须跟着发生变化；另外如果外部应用和服务之间存在一个长请求(long-running request)，那么客户应用就会发生堵塞，即使通过多线程可以解决客户应用堵塞问题，但服务要实现双向通讯，这不仅增加开发成本，而且会影响服务的性能和水平扩展。

消息事件为我们提供了另一种通讯机制。其实在现实生活中，我们大部分的行为并不是被某人或某物直接驱动(RPC就是由客户代码主动驱动)，而是被某一事件(消息)触发(事件驱动)。比如当我们听到电话铃声，我们会主动的去接起电话，而不是被电话控制着去接电话；当出租车司机接收到网约订单，他们才会主动去接客户。这些主体和目标对象并没有直接发生联系，每一个责任主体都是主动做自己责任范围的事。通过消息驱动，我们可以**解耦**服务与服务之间的依赖，这非常有利于实现服务自治。

因为服务之间没有依赖，所以客户端不需要等待服务给出响应就直接返回，这种**异步的通讯**方式不仅有助于提高系统的用户体验，还能减少服务端的通讯压力。一些不需要实时响应或服务暂时超负荷的情况下，我们可以将消息保存到消息队列中，等消息被服务处理完，再发通知告诉客户应用，我们也将这种方式叫做生产者与消费者模式。它和面向对象设计模式中的[观察者模式](https://qunxi.github.io/2017/08/27/design-pattern-observer.html)有着异曲同工之处。下图是RabbitMQ的消息订阅和处理的示意图：![message queue](https://www.rabbitmq.com/img/tutorials/intro/hello-world-example-routing.png)

如上图所示，我们需要一个独立于服务的消息代理(Message Broker)，它负责将消息从发送端传送到接受端，在此过程中它可以对消息进行路由、编排、转义和持久化等。有的消息代理甚至实现了一些与API网关类似的功能(负载均衡、服务发现、服务解耦等)，而且在架构上它也和API网关一样具有隔离层的效果。目前网上有很多开源、成熟的消息中间可用于商业项目(比如[Kafka](https://kafka.apache.org/)、[RabbitMQ](https://www.rabbitmq.com/)、[ZeroMQ](http://zeromq.org/)等)。虽然基于消息代理的开发非常的方便，但是我们依然要注意它的使用，因为我们很容易用着用着就将它们扩展成为一个重量级的ESB(Enterprise Service Bus)，这不但会引导系统设计趋向中心化，而且很容易增加消息管理的复杂度。这也是为什么MartinFowler提倡在微服务设计中应尽量做到**Smart endpoints and dumb pipes**。

我们通常认为消息是一种转瞬即逝的事件，所以我们更关注消息发生的那一刻，对于已经发生的消息我们并不太留意。其实消息是一种已经发生过的**事实**，如果我们能够保存所有发生过的事实，那么我们就可以追溯事物每个时刻的状态，甚至可以还原某一刻的**世界**，是不是很神奇。在分布式的世界，要做到全局实时的一致性几乎很难做到，根据CAP原则，我们必须在一致性、可用性和分区容错之间做出权衡。如果我们持久化了所有的消息，至少我们可以保证最终数据的一致性(银行很多业务就是这么实现的)。所以将通过领域设计，抽象领域事件(Domain Event)是基于消息驱动非常好的实践方法。如果感兴趣的可以参看领域驱动开发的红宝书([Implementing Domain-Driven Design](https://book.douban.com/subject/11940943/))。

# 后记

本文介绍了服务间通讯的基本概念，具体的实现和设计并不是那么简单，需要综合考虑很多问题，而且这里介绍的通讯是基于服务开发这个层面，在后续的服务部署中，我们也会涉及到服务的管理和编排，这些过程也需要考虑服务的通讯，我们会在后续的章节继续讨论这些问题。