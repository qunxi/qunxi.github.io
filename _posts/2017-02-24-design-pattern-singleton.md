---
author: qunxi
create: 2017-02-24 18:43+08:00
update: 2017-02-24 18:43+08:00
layout: page
title: "设计模式-Singleton模式"
description: ""
comments : true
categories:
tags:
- 设计
- 模式
---
# 前言

如果在开发人员中做个调查，我相信Singleton模式应该是最被程序员所熟知的一种模式。这里主要的原因可能是Singleton模式实现相对简单，使用意图也很明确。当然也是因为这些原因，也导致了它容易被滥用。在我看来Singleton模式被使用的场景不应该那么普遍，甚至使用的越少越好。这篇文章我们就聊聊Singleton模式。
<!--more-->

# 1. 创建唯一的实例

和[**工厂模式**](https://qunxi.github.io/2017/02/18/design-pattern-factory.html)一样我希望通过一个具体的场景开始讨论，然后开始一步步的重构，这样可以让我们对模式的意图和作用更加容易理解。下面我们就以日志（Logger）为例开始讲解，日志（Logger）是开发人员非常熟悉的功能，通过日志我们可以记录软件运行时可能发生的问题或重要事件，当然在一个系统中，像日志这样经常使用`Cross-cutting`对象，往往只需要一个实例，而且我们会在软件的不同层次访问它。如果用[**工厂模式**](https://qunxi.github.io/2017/02/18/design-pattern-factory.html)实现上述场景，我们也许会这样做：

```
class Logger{
    public Logger(){}
}

class LoggerFactory{

    private static Logger logger;

    public static Logger Create() {
        if(logger == null){
            logger = new Logger();
            return logger;
        }
        return logger;
    }
}
```
我们通过工厂`LoggerFactory`的`Create`方法来创建Logger对象，而且每次调用`Create`方法时判断Logger对象是否已被创建，这样确保了通过`LoggerFactory`只会生产一个Logger对象。但是严格意义上讲这并不能完全禁止只实例化一个对象，因为我们可以在任何的地方直接通过`new`来实例化`Logger`对象。那如何限制全局的唯一性呢，我们来看看Singleton模式的定义。

# 2. Singleton模式（单例模式）

> **保证一个类仅有一个实例，并提供一个访问它的全局访问点。**

从定义看Singleton模式应该包含两个条件，首先要保证一个类只能有一个实例，然后还要保证只有一个全局的访问点。上面例子并没有满足任何一个条件。如何做到只实例化一个对象？首先我们必须使用`private`或`protected`封装类的构造函数，否则它将会在任何地方被调用。如果构造函数已经被封装，那么唯一可以访问它的地方就是类本身了。我们对之前的代码做了点小小的重构，将`LoggerFactory`修改成`Logger`，然后把构造函数设置成私有或保护成员，最后我们将`Create`静态函数重命名为`Instance`。代码重构如下

```
class Logger{
    
    private static Logger logger;

    private Logger(){ }

    public static Logger Instance(){
       if(logger == null){
           logger = Logger();
           return logger;
       }
       return logger;
    }
}
```
![singleton pattern](/post-images/2017_2_24_singleton.png)
上面的代码利用静态成员方法调用私有构造函数和判空来保证实例全局的唯一性。当然我们要保证任何情况都是唯一的，就必须考虑多线程的情况，所以这里我们要保证`Instance`这个静态函数是线程安全的。下面我们对上边的代码做进一步重构：

# 3. 多线程下的唯一保证

上面的代码如果运行在多线程的环境下很可能构建多个实例，这对于C#来说可能只是暂时的消耗了一些内存和性能，但是对于C++来说这个可能是个严重的内存泄露。对于多线程的实现其实也很简单，我们只要在`if(logger == null)`的外部加一个同步锁，就可以防止多线程切换上下文时出现的**`Race Condition`**，而构造多次实例。当然加锁在性能上是有损耗的，比如在对象已经实例化完成后，仍然去同步访问`Instance`方法而付出的性能损耗是没有必要的，所以我们可以通过`Double-Check Locking`进一步优化。下面便是优化后的代码。

```
class Logger{

    private static readonly object syncLock = new object();

    private static volatile Logger instance;

    private Logger(){ }

    public static Logger Instance(){
            if(instance == null){
                lock(syncLock){
                    if(instance == null){
                        var temp = new Logger();
                        Interlocked.Exchange(ref instance, temp); //避免编译器优化带来的现场不安全
                        return instance;
                    }
                }
            }
            return instance;
    }
}
```
上面我们使用同步锁的方法解决了多线程创建单例的问题，当然我们还可以利用C#自有特性，在不需要同步锁的情况下完成同样的效果，因为.Net保证了静态成员初始化是线程安全的。这种方式不是所有语言通用的，所以我同时将静态函数改成了静态属性，目的仅仅想告诉你这种方式只适合C#。

```
class Logger{
    private static readonly Logger instance = new Logger()

    private Logger(){ }

    public static Logger Instance{
        get{
            return instance;
        }
    }
}
```

# 4. 懒汉模式和饿汉模式

有时我们将Singleton模式的实现方式分为**懒汉模式**和**饿汉模式**。我们将第一次调用时才初始化实例的实现方式叫**`懒汉模式`**，否则就叫**`饿汉模式`**。首先我必须强调上面所实现都是**饿汉模式**。如果你有C++的开发经验，也许会对最后一种实现有所疑惑，这里必须指出C#的静态成员变量初始化是在类第一次调用`Instance`时完成的，而不像C++是程序一开始运行时完成的。

当然提到静态对象初始化，C++也可以分别利用静态成员变量和静态局部变量来实现Singleton模式的**懒汉模式**和**饿汉模式**，请参看下面代码。但是我不推荐使用这种方式，首先C++的静态成员变量初始化是在程序运行初期就开始构造了，那么这会导致没有必要的开销，因为我们构建的对象可能永远也不会被调用的；其次静态成员的初始化顺序是不确定的，如果`Logger`对象的构造还依赖于其它对象，而这些对象又在它初始化之后，那么就会出现不确定性。最后C++保证静态局部变量初始化的线程安全是依赖于编译器的，不同编译器实现也不同(目前VS2013不支持，据说C++11标准已经要求静态初始化程序线程安全)。基于以上不确定性，我们不推荐下面C++的实现方式。

```
// 饿汉模式
class Logger{
    public：
        static Logger getLogger(){
            static logger;
            return logger;
        }
    private：
        Logger(){};
}

// 懒汉模式
class Logger{
    public：
        static Logger getLogger(){
            return logger;
        }
    private：
        Logger(){};
        const static Logger logger;
}
```

# 5. Singleton模式的思考

* ## 模板

曾经遇到类似下面用模板实现的单例模式，它将需要构建的对象类型作为模板参数。

```
class Singleton<T> where T : new()
{
    private static readonly object syncLock = new object();
    
    private static T instance;
    
    public static T Instance{
        get{
            if (instance == null){   
                lock(syncLock){
                    if(instance == null){
                        instance = new T();
                    }
                }	
           }
           return instance;
        }
    }

    private Singleton() { }
}
```
先不说这样实现是否正确，首先使用模板的目的就是能复用这种模式，但在现实项目中真的有多少场景需要使用Singleton模式吗？其次因为模板类型并没有封装构造函数，所以我们可以跳过这个Singleton自己去`new`一个对象。这个和一开始的例子并没有什么差别。也许我们可以通过文档来告诉开发人员必须通过Singleton模板类型去构建对象，但能在代码上限制构造，会比文档更高效，毕竟文档的制约性不在文档，而在人的主观意识。

* ## 全局访问

Singleton是利用静态成员来实现的，虽然它有类作用域的访问限制，但因为它提供了全局访问点，所以它有和全局变量一样的弊端，因为上面的例子相对比较简单，而且都是读操作，所以好像没有什么大问题。如果我们有一个复杂的单例对象，它需要维护更新许多状态，而且这些状态还和不同的场景有一定的依赖关系，那么这时出了问题，就会很难定位，因为你不知道什么地方修改了某个状态会影响当前问题。所以使用Singleton模式一定要谨慎，我的建议不要因为性能、方便访问亦或简单等理由使用单例模式，只有真正符合单例模式定义的场景我们才使用它。 

* ## 内存管理

C#可以通过垃圾回收机制来完成内存的管理，但是GC什么时候回收我们是无法控制的，会不会长时间不引用导致对象被回收，最后导致不断的重建对象呢，这点其实不用担心，因为单例通过静态成员实现，而静态成员的持有者是类，它的生命周期是和`AppDomain`关联，所以一般情况没有必要太担心GC回收问题。虽然一个单例对象不应该频繁的构建和删除，但是对于一些消耗资源比较大的对象，如果我们知道它具体在什么时候销毁，提供一个负责销毁对象的接口是非常有必要的。C++我们只要添加一个静态函数去`delete`对象即可，C#可以通过实现`IDisposable`接口。最后我还是要强调一点，如果在你考虑实现手动销毁对象操作时，你最好考虑下如何使用其他方式替代Singleton模式。

* ##  反射

很多语言因为支持反射，已经不能仅仅利用Singleton模式来控制实例化的个数。因为反射机制的出现，已经使对象的构建不仅仅依赖构造函数，比如下面C#代码的实现。

```
Type type = typeof(Logger);

System.Reflection.ConstructorInfo[] ConstructorInfos = 
       type.GetConstructors(
                   System.Reflection.BindingFlags.Instance | 
                   System.Reflection.BindingFlags.NonPublic);

Debug.Assert(ConstructorInfos.Length == 1);

var obj = ConstructorInfos[0].Invoke(null);

Logger logger = (Logger)obj;
```
当然一般情况下我们并不会那样实现，这里只是提出了另一个角度的看法，目的想告诉大家任何的模式或方法都有其局限性，应该根据具体问题具体分析。

* ## IoC容器

IoC容器虽然不能阻止对象实例化的个数，但是它可以`Register`某些对象，使其在容器内是唯一的。容器我们一般在程序启动时注册好需要的对象，然后在适当的时候`Resolve`它。也许你会问这和开始的工厂方法有什么不同！首先我觉得Ioc容器可以更好的解耦构建对象与客户代码的关系，同时也不用引入对工厂类的依赖。所以一般情况我更建议使用容器。

# 后记

**模式**这个词本身就带有固定，不变的含义，所以很多时候，大家会像套公式一样去使用它，但是千万不要反过来用模式去套场景（因为很多人开发项目并不一定都能遇到合适的场景去应用模式，有时为了show off而在项目中不成熟的使用它）。最后还是再说一句使用Singleton要谨慎。


