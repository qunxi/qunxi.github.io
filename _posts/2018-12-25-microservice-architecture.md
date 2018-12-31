﻿---
author: qunxi
create: 2018-12-25 18:43+08:00
update: 2018-12-31 18:43+08:00
layout: page
title: "微服务架构"
description: ""
comments : true
categories:
tags:
- 基础
---

# 前言

微服务已然不是什么新名词，但是它依旧是一个时尚的名称。虽然它不像区块链技术那样开拓出一块新的领域，但至少在架构领域它在不断的扩大着自己的影响力，并被越来越多的人所接受。在18年的最后一个月我想谈谈自己对微服务的理解。
<!--more-->

# 1.为什么我们需要微服务

曾经问过一些开发，为什么我们要使用微服务架构？得到的大部分答案是，因为它非常的强大，很多大公司都在使用，而且许多开源框架以及前沿技术支持这样的架构开发。听上去很有道理，但是这些仅仅是开发人员的自我感受，而不能作为技术选型的依据。不结合公司基本情况以及业务的技术或架构，只能是纸上谈兵。技术或架构的选型一定不是这个技术或者架构多么流行，而是这个技术或者架构能否给公司长期利益带来优势，并且能解决公司现有架构中的一些突出问题。

回到问题的本身，我们(这里并不包括所有软件行业，而是目前软件业的大环境)现在遇到了什么问题？首先我们需要解决的问题变的越来越复杂，从而导致系统本身的复杂度变高；其次行业竞争压力变大，软件开发周期需要不断的缩短；另外用户对软件的体验以及在可用性和正确性的要求越来越高；这些都是大规模企业软件架构中必须面临的问题。归根结底就是现在我们需要一种既能**快速迭代**，又能**可持续发展**的架构来支持我们庞大而复杂的业务。

我们的产品是否遇到了类似上面的问题？而微服务真的能解这这些问题吗？在给出答案之前，我们至少要回答下面一些问题：什么是微服务？微服务给我们带来了什么？微服务真的适合我们吗？

# 2. 什么是微服务

什么是微服务？顾名思义就是非常小的服务。但是这个小到底多小才能算得上是"微"？而"服务"在不同的上下文也有不同的意思，那么这里的服务又是指的什么？到目前为止这个定义仁者见仁，智者见智。这里引用[Martin Fowler在他的博客](https://martinfowler.com/articles/microservices.html)中给出的定义：

> 微服务架构就是以一系列小型服务的方式开发一个独立的应用系统。其中每个服务都运行在自己的进程中，通过利用类似HTTP资源API这样轻量机制进行通信。这些服务围绕业务能力进行构建，并能够通过完全自动化的方式进行独立部署。这些服务可以使用不同的编程语言以及不同的数据存储技术来实现，对这些服务我们应该以最低限度的中心化管理。

首先定义解释了服务这个概念，他强调了微服务是以**业务能力**划分，可以**独立部署**并且利用**轻量级机制进行通讯**的独立进程。因为服务相互独立，所以每个服务可以使用自己的技术栈；因为服务间的通讯平台无关，所以一个系统可以由多种异构服务组成；因为服务可以独立部署，所以去**中心化管理**可以保证系统的稳定性。定义并没有明确的指出如何度量服务的大小，因为大小都是相对的，简单的通过一个固定标准(代码行数，模块数目或团队大小)来划分显然不够科学，决定是否应该独立出一个服务的可行方法是-清楚的划分出业务能力以及在这个业务范围内实现服务自治的边界。在这个边界内，服务只处理一个明确业务，这对开发人员来说是非常有吸引力的。因为开发人员可以在很快的上手之，而不用预先学习大量和当前问题不想关的业务，其次开发人员也不用担心自己的修改给整个系统带来影响。

微服务架构和之前出现或流行过的架构有什么不同？尤其是每当我们提到微服务，我们就会涉及单体(Monolith)架构和面向服务架构(SOA)，它们之间的联系又是什么？我想只有清楚了它们之间的关系我们才能更清楚什么是微服务以及为什么我们需要微服务。

# 3. 单体应用(Monolithic Application) vs 微服务(Micro Service)

每当我们提到单体应用，我们第一反应就是庞大(不仅代码量还有业务复杂度)；第二反应就是这是一个具有历史的产品或系统；第三个反应就是它非常的脆弱，在系统面前所做的一切都必须带着"敬畏之心"(是否有把玩古董的心情)。显而易见，单体系统不仅系统复杂度高，而且模块之间相互耦合，任意的修改都会影响到整个系统，甚至影响团队间的合作以及功能的部署。现在我们不能事后诸葛的说为什么我们不能一开始就使用微服务架构？因为任何事物的发展都是和每个时代背景有关，架构亦是如此。

不同的架构都是由当时的时代环境所孕育，过去享受着摩尔定律的便利以及并不复杂的业务场景，单体架构不仅开发简单，而且可以通过硬件升级来提高软件的性能。但是随着软件复杂度的不断上升，以及摩尔定律所遇到的瓶颈，单体架构的弊端开始显现。随着云平台的出现，为了充分利用云优势，软件架构需要更好的水平扩展能力，而单体架构的水平扩展对于硬件的要求非常严格，它不但不能充分利用大量小的计算资源扩大软件规模和提升计算能力，而且部署云的成本也非常高，所以基于微服务的架构应运而生。

虽然单体应用有其自身的缺陷，但是我们不能完全否认它所沉淀下来的理论和实践。微服务依然可以使用单体的分层架构来实现，我们需要注意的是在合适的时候对已有的业务能力进行拆分。如果想利用微服务实现一个新产品，我们依然建议你从单体架构开始，然后随着业务复杂度的增加，进行微服务拆分。因为微服务的初始效率并不比单体架构高(参看下图)，而且在新产品开始之初，我们对产品的熟悉程度不足，贸然拆分成多个服务，而且服务间的边界并不正确，反而会增加服务间协作成本。如果我们是在重构遗留系统，利用领域驱动开发(DDD)中的Bounded Context作为划分服务的边界是一种相对科学的方法，因为领域驱动开发就是着眼于业务能力进行领域建模，而这正是和微服务的宗旨非常相近-基于业务能力构建服务。

![mono vs ms](https://martinfowler.com/bliki/images/microservice-verdict/productivity.png)

# 4. 面向服务架构(Service Oriented Architecture) vs 微服务架构(Micro Service Architecture)

提到微服务，就会有人想到面向服务架构(SOA)，甚至很多人将SOA作为微服务的对立面，认为SOA是种反模式。那么SOA到底实什么？和微服务一样，到目前为止SOA也没有一个标准定义，所以不同的企业使用不同的方式实现着自己理解的SOA，有人利用SOA作为企业应用的集成和整合，有人利用SOA作为组件模型来进行应用的拆分，还有人将SOA认为是利用RPC作为接口的服务调用。正是因为不同的理解，所以一些人认为微服务架构是一个全新的独立于SOA的架构。而我认为微服务仅仅是SOA的一种实现方式，或者是当前背景下的一种正确实现SOA架构的方式。它在弱化以前SOA架构中的中心化思想(包括数据、服务集成等)。它强调服务自治、数据自治，提倡以RPC或简单消息队列作为服务通讯，它更强调的是业务能力，而不仅仅是简单的系统集成。微服务只是SOA架构的拓展与演进。

我们依然需要考虑服务的管制，我们依然需要高效的RPC通讯，我么依然需要考虑基于事件驱动的异步模型，我们依然需要考虑如何做服务的集成等，也许我们的具体技术或方法会有所不同，但是我们一直站在SOA这个巨人的肩膀上不断的创新和进化。

# 5. 微服务我们准备好了吗？

现在可以回答为什么我们要使用微服务架构了吗？貌似答案已经很明显。但是如果一个问题那么显而易见，那它一定存在我们还不知道的陷阱。我们真的做好了使用微服务的准备了吗？在光鲜亮丽的优点下面我们到底隐藏了什么？也许是像这样密密麻麻的服务。![service choas](https://pic2.zhimg.com/80/v2-f025c8dbcbf925ec9a9874ace9a41599_hd.jpg)

* 服务的拆分：微服务不是简单将服务拆分的越小越好，系统的复杂度是所有服务的累加，随着拆分粒度的减小，系统集成的复杂度就不断的攀升，同样拆分的粒度越大，单个服务的复杂度就会上升，如何权衡不是简简单单的0或1的问题。架构或资深开发必须对业务边界有清晰的划分，否则服务间不必要的耦合将导致服务间协作成本大大增加。这些你们准备好了吗？

* 去中心化：微服务的去中心化，不仅解耦了服务间的依赖，而且保证了系统的稳定性。但伴随而来的是开发的复杂性， 开发人员不仅仅要保证单个服务的正确性，好要确保整个系统在分布式环境下的(事务、数据等)一致性问题，另外进程间的调用问题(稳定性、性能、失败处理)也远远比进程内调用复杂的多，还有一个独立部署的微服务往往由一个团队维护或者开发，这就需要团队成员具备一定的全栈能力。开发你们做好准备了吗？

* 服务治理：微服务的拆分降低了单个服务的复杂度，但是系统的能力是所有服务相互协作的结果，我们可以通过手动的方式编排或部署一个简单的系统，但是一旦服务数量达到一定的数量级，手动部署、编排将是一种恶魔。**自动化**部署、编排将是必然的趋势，服务发现、路由，服务容器化等这些非功能性需求由谁来实现？我们的基础架构支持这样的扩展吗？另外服务数量会随着时间不断的增多，监控系统中每一个服务的状态是非常有必要的，这些都是扩容、限流、调试的基础，**可视化**系统中服务的状态和关系是系统运维非常必要的手段。Devops你们准备好了吗？

* 服务测试：微服务给单个服务的测试带来了极大便利，我们可以在每个服务中做大量的单元测试，但是集成测试、端对端测试将不再像单体应用那么简单。我们要考虑的问题不仅仅是功能的准确，还有考虑分布式系统中各式各样的非功能性问题？性能问题、系统稳定性、网络问题、以及安全问题等等。测试你们做好准备了吗？

* 组织架构：即使我们具备了上面的技术能力，那么我们就一定能实现一个完美的微服务架构吗？[康威定律](https://en.wikipedia.org/wiki/Conway%27s_law)指出一个企业的组织架构将会影响企业产品的架构，反之亦然。如果团队的划分不合理，这些不合理会体现到产品的设计中，而且影响团队的开发效率。公司已经做好了组织架构调整的准备了吗？

综上所述，再回到开始的问题，我们为什么选择微服务？我希望这时每一个人都有自己不同的答案。微服务适不适合你的产品以及你的团队，只有你们自己最清楚。

# 后记

微服务架构和其他架构一样，它有它适合的场景或条件。它不是所有问题的标准答案。但是我相信作为一个合格的架构师或者资深开发人员，我们应该确保我们所做的设计或架构要尽可能的在业务上做到解耦，而且适合演进式发展。这样即使现在也许微服务不适合我，但是等到了适合的时候我们可以很方便的往上面迁移。退一万步讲，哪天微服务退出了历史舞台，我们是否依然可以非常简单的将现有业务能力迁移到新的架构上，这才是我们应该更加重视的问题。