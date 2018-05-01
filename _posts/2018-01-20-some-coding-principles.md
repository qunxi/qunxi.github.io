---
author: qunxi
create: 2018-01-10 18:43+08:00
update: 2018-01-10 18:43+08:00
layout: page
title: "程序员应该知道的一些原则"
description: ""
comments : true
categories:
tags:
- 设计
- 模式
---
# 前言

每一种模式都是解决对应问题的“套路”，但是有**术**而不求**道**，最后不是按图索骥，就是到处炫技。技巧千千万，但万变不离其中的还是那些更为“普世”的原理。这篇文章我们一起来聊一聊程序员应该知道的一些原则。
<!--more-->

## S.O.L.I.D原则

SOLID是由Robert.C Martin引入的设计原则，它由五个具体的原则组成:

* 单一职责原则（SRP: Single Responsibility Principle）：

> 就一个类而言，应该仅有一个引起它变化的原因。

* 开闭原则（OCP: Open & Close Principle）：

> 软件实体（类、模块、函数等等）应该是可以扩展的，但是不可修改的。

* Liskov替换原则（LSP: Liskov Substitute Principle）：

> 子类型必须能够替换它们的基类。

* 接口分离原则（ISP: Interface Separation Principle）：

> 不应该强迫客户依赖于它们不用的方法。

* 依赖倒置原则（DIP: Dependency Inversion Principle）：

> 高层模块不应该依赖底层模块。二者都应该依赖于抽象；抽象不应该依赖于细节，细节应该依赖于抽象。

SOLID原则对于很多开发者并不陌生，在之前的[文章](https://qunxi.github.io/archive/)中我也有提及其中的一两个原则。下面我们简单的来聊一聊每一个原则:

* SRP很容易被它的名字所误解，其实它所谓的职责强调的是引起一个类或模块的变化原因，而不是简单的行为或功能（虽然大部分情况不同的功能可能会导致不同的变化，但是如果多个功能导致的变化是一致的那么它也符合SRP原则）。为什么要控制变化点？如果每个变化都被隔离而不是耦合在一起，那么我们的修改都是独立的，并且不会相互影响。可以想象如果变化错综复杂，盘根错节，那么要维护好这些代码会多么困难。

* OCP强调的是对扩展要开发，对修改要封闭。具体的例子可以可以参看[抽象工厂和简单工厂](https://qunxi.github.io/2017/02/18/design-pattern-factory.html)，简单工厂对于创建新对象必须修改已有的代码，而抽象工厂只需要重新派生一个新类型，而以有代码并不需要做任何修改。为什么我们倾向扩展而不是修改？因为扩展的复杂度变化是线性的、符合预期的（我们需要做的只是依样画瓢），而修改的复杂度是不确定的，它取决于依赖的复杂度（修改也说明了变化是在设计预期之外的，我们要重新梳理原有逻辑重新设计）。

* LSP强调的是抽象能力，好的抽象是基类完全可以替换成子类，而且不会导致异常。为什么抽象出一个合理的基类如此重要，答案很简单-降低我们的维护成本，设想如果我们的代码都是依赖抽象，那么我们唯一要做的就是在初始化时生成一套应的具体子类，剩下的逻辑框架完全可以复用，否则我们需要处理不同条件下的复杂逻辑（这点也印证了OCP）。 LSP可以很方便的验证我们抽象是否合理，比如我们经常利用常识来抽象我们的类关系，但是很多时候其实并非合理，`Rectangle`和`Square`就是很好的例子，我们知道正方形在数学上属于特殊的矩形，如果将矩形和正方形设计成父类与子类的关系，那么SetWidth和SetHeight的矩形接口，对正方形来说感觉会很奇怪。

* ISP强调的是接口的内聚与解耦，相对其他原则，它更容易被理解，设想如果一个对象依赖一些自己更本不需要的接口，那么即使其中一些不相关的接口发生变化，那么这个对象也需要重新编译，毕竟接口的变化成本非常大。

* DIP的定义我们已经在很多地方看到，我们就不再赘述，这里我们来聊聊什么是**依赖倒置**，通常软件的依赖都是高层模块依赖底层模块，但是很多软件的核心模块其实是软件领域模块，比如说业务模块调用底层数据访问模块，很明显我们应该抽象业务的接口，然后数据模块实现它，提供相应的数据，否则一旦数据库发生变化就会导致上层业务模块也要跟着一起变化。还有操作系统我们需要依赖驱动程序去控制硬件，那么到底是操作系统依赖驱动程序还是驱动程序依赖操作系统的统一的抽象接口呢？我想答案不言而喻。

## DRY（Don't Repeat Yourself）

> DRY的直译就是不要重复你自己。

这里的Yourself指的是代码。代码重复最大的问题是变化点的扩散，一旦重复代码的逻辑发生变化，那么所有依赖的代码、模块都需要修改、重新编译并重新部署。重复的地方越多维护的成本代价越高。从程序员的角度最容易犯违反这一原则的就是肆无忌惮的使用Ctrl-C和Ctrl-V，而常见的解决代码重复的方法有：函数、类或模块的提取，使用继承或则组合的方式，或使用模板编程等复用已有逻辑。这条原则强调的是提高代码的可重用性。

## 迪米特法则（Law of Demeter）

> 迪米特法则（Law of Demeter）：一个软件实体应当尽可能少的与其他实体发生相互作用。每一个软件单位对其他的单位都只有最少的知识，而且局限于那些与本单位密切相关的软件单位。

迪米特法则主要强调的是细节的封装，我们应该尽可能少的暴露实现的细节，而且以接口的形式提供仅限于客户代码需要的功能。最常见的方式就是在一开始设计类或模块时，尽可能的将所有的成员方法、函数都设置成`private`，只有在确实需要暴露的情况下才设置成`public`访问权限，另外尽可能的少或不使用全局变量或对象。最形象概述这条原则的是[代码的设计](https://qunxi.github.io/2017/01/30/object-oriented-design.html)中附带的冰川图片。

## Hollywood原则（Don't call me，I will call you）

> 不要打Call（调用）给我，等我有需要的时候自然会Call（调用）你。

Hollywood原则强调的是反转控制以及关注点的分离。在SOLID原则中我们提及的**依赖倒置**也是Hollywod原则的体现，尤其在设计框架时，框架不会依赖于我们的业务代码，它只提供接口，我们的业务模块实现接口，框架会在合适的时候去调用业务代码。如果你使用过IoC容器就更能体会什么叫Hollywood原则，我们会先注册对象到容器中，客户代码根本不需要知道它何时生成，如何构建，我们通常结合DI（依赖注入）构造所需要的对象。Hollywood原则的另一个应用就是回调函数或event（了解event请参看[C#委托(Delegate)](https://qunxi.github.io/2016/10/16/csharp-delegate.html)），我们只需要预先定义好函数或注册事件，而真正调用或触发event的时机则由主体对象决定（观察者模式）。

## KISS原则（Keep It Simple And Stupid）

> 保持事情的简单和傻瓜

无论是设计还是实现，尽可能的保持简单、易懂。理由很简单，越是简单的问题，越容易解决；越是简单的解决方案，我们管理维护的成本就越低。但是简单的范畴又很难定义，因为个体的差异性或领域的特殊性，都可能导致不被一些个体理解的复杂问题，对于另外一些个体来说可能是简单的；另外并不是所有复杂问题都能找到简单的解决方案。所以说简单是一个相对的概念。既然我们不能定义绝对的简单，但是我们可以找到我们不擅长的问题，这些问题少了，那么整个系统就会变得相对简单。软件开发中耦合（依赖）、细节、变化都是我们开发过程中经常遇到的头疼问题。我们可以通过封装来隐藏细节，让用户通过他们可以理解的接口去调用更复杂的系统；我们可以通过封装，解耦去阻止变化的传递、扩散，当然我们可以通过抽象去分离交织在一起的变化点。另外有些复杂度并不是来自问题的本身，而是我们自己对复杂度的迷恋，这点我们将在下面的YAGNI原则中进一步讲述。

## YAGNI原则(You Aaren’t Gona Need It)

YAGNI原则是Kent Beck在XP（极限编程）中提出的概念。它主要诟病的是过度设计（overdesign）或过度实现，我们总是假设我们的额外功能客户一定会非常喜欢；我们总是假设在未来的某一天这段代码可能会被用到；我们总是假设我们可以预先为将来的功能做好充分的设计。对不起，我们对自己的预测能力过于的自信，如果有那种能力，那么我们还有什么理由抱怨需求变化为什么那么快，客户为什么在抱怨他们想要的和我们实现的不一样。还有一些复杂度并不是系统自身带来的，而是我们在不知觉中利用制造复杂度来满足自己的虚荣心（Show Off）。原本可以很容易解决的问题，我们非要用一些突兀的技巧来炫耀我们的所学，最后是抱起石头砸了自己（或队友）的脚。实践规则：我们不可能一次性将问题解决的完美，我们应该通过不断的迭代中去重构接近这个目标，我更倾向于问题出现第三次时我再开始考虑复杂的设计。

这里必须在强调一次，我们反对的是过度设计，而不是不需要设计，设计的本身就带有复杂度，但是设计本身就在平衡与协调复杂度，这里可以参考**2/8原则**。

## 2/8原则

2/8原则最早出现在经济学领域，后来衍生到社会学、管理学甚至工程学。在软件开发中这些现象也非常普遍，我们会发现80%的问题（bug）经常反复出现在20%的代码中，我们会发现客户经常使用的永远是那20%的功能，甚至你会发现，我们真正写代码的时间其实只占到的20%，更多的时间可能花费在了会议、邮件、交流等问题上。2/8原则告诉我们，在做任何事情前，我们应该抓住主要的问题和矛盾，把真正的精力放在最主要的事情上，而不是抓了芝麻丢了西瓜。

## 破窗效应

> 此理论认为环境中的不良现象如果被放任存在，会诱使人们仿效，甚至变本加厉。

破窗效应其实是一个心理学效应，但是我认为对程序员非常重要，我们总是给自己找理由，我先将功能实现，然后再回来clean code。其实这些都是自欺欺人，事情往往会一点一点变糟，然后变得连你自己都不想维护它，最后习以为常到已经不能分辨什么是好的代码，什么是坏的代码。发现有“味道”的代码，不要犹豫马上行动，千万不要等变成了“焦油坑”，让自己深陷的无法自拔，那时已为时晚矣。

# 后记

其实上面提及的所有原则也并非最终的“道”，所谓道可道非常道，它们更像是具体模式的抽象。而且因为篇幅的有限还有其他一些原则作为开发者也应该了解，比如CRP(Common Reuse Principle)、REP(Reuse/Release Equivalence Principle)、CCP(Common Closure Principle)等，如有需要可以参看Robert.C Martin写的《敏捷开发-敏捷软件开发：原则、模式与实践》。