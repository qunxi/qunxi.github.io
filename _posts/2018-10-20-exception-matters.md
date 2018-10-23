---
author: qunxi
create: 2018-10-20 18:43+08:00
update: 2018-10-21 18:43+08:00
layout: page
title: "异常处理"
description: ""
comments : true
categories:
tags:
- 基础
---

# 前言

异常处理是面向对象语言中很常见的特性，但是在日常开发过程中，经常发现很多开发人员要么重不使用异常，要么就简单粗暴的捕获一个通用异常替代所有异常。今天我们一起聊聊异常处理(这里我们以C++和C#的异常作为例子)。
<!--more-->

# 1.异常(Exception) vs 错误码(Error Code)

为什么我们需要异常，在异常之前我们是如何处理错误的？这我们需要从C语言开始说起，C语言是没有异常这个概念的，所以以函数返回值来返回错误是最好的方法。我们可以通过返回true/false表示函数执行成功或失败，如果要返回超过两种以上状态，我们可以用枚举(enum)来声明错误的类型，调用代码根据返回错误类型做出相应的处理(状态恢复，通知用户或清理资源)。参看下面的代码：

```C
enum ErrorCode{
    Success,
    Failed,
    NoAuthorized,
    Unknown
};

Response accessDashboard(Request request);

int main(){
    Request request = makeRequest();
    Response response = accessDashboard(request);
    switch(response.ErrorCode)
    {
        case ErrorCode.Success:
            // 处理返回结果
            response.convert(); //#1 这里convert也可能发生错误，它的错误处理类似，这里我们将其忽略
            break;
        case ErrorCode.Failed:
            // 处理访问错误
            break;
        case ErrorCode.NoAuthorized:
            // 处理授权失败
            break;
        case ErrorCode.Unknown:
            // 记录未知错误，并通知管理员
            break;
    }
    .....
}
```

这样的处理再平常不过，而且也可以很好的应用在像C++或C#这样的面向对象语言中，但是为什么需要权限的引入异常这个概念呢？首先，在语言特性方面，像C++引入了构造函数、引用以及一些重载作用符等面向对象新的特性，而C#引入了像事件、属性等这样一系列语法糖，如果以返回错误码的方式完成错误处理，会使代码变得晦涩难道，不易维护。比如构造函数是没有返回值的，还有像dynamic_cast<MyObject&>只能返回一个对象的引用。(如果构造失败或者类型转换失败，我们不可能返回另一个数据类型)。所以在语言方面需要引入异常这个概念。在代码设计使用方面，异常也给我们带来更直观，清晰的实现，同样先看看下边关于异常的使用：

```C++
class FailedException{};
class NoAuthorizedException{};
class ConvertFailedException{};

int main()
{
    try
    {
        Request request = makeRequest();
        Response response = accessDashboard(request);
        response.convert();
    }
    catch(FailedException& ex){
        // 处理访问错误
    }
    catch(NoAuthorizedException& ex){
        // 处理授权失败
    }
    catch(ConvertFailedException& ex)
    {
        // 处理转换失败
    }
    catch(std::exception& ex){
        // 处理一些标准库中抛出的异常
    }
    catch(...){
        // 处理未知的异常
    }
   .....
}
```

从代码的结构上，两种实现没有太大的区别，就是将C语言的switch语句换成了try-catch。但如果我们将C语言#1处的错误处理展开，你就会发现为什么异常处理会比错误码返回更加科学和直观。在C语言的代码中我们会发现业务代码和错误处理的代码交织在一起，这不仅阅读起来非常不方便，而且不易维护和扩展。而C++代码则不同，异常处理和业务逻辑完全分离。另外对于调用端的开发者，有了异常我们不需要知道被调函数所有可能发生的错误。因为我们可以通过catch(...)捕获未知的异常(当然我不建议这样做)或通过抛出异常提醒我们那些可能被我们忽略的异常。

当然这些易用性和直观性是需要代价的，就像C#和C++一样，C#对于开发更加友好，而这些优势又需要在性能或灵活度上做出牺牲，异常也是如此。首先，异常类比错误码需要维护更多的异常信息(调用栈)以及内核对象的支持。其次异常一旦抛出，异常对象可能被拷贝而且会将调用栈展开，这些都是需要消耗性能成本。所以这些都会是程序变大，运行是性能损耗。所以如果你的程序真的很在乎性能和实时性，那么你应该直接使用C(如果你说C#怎么办，我只能说，你都使用C#了你还在乎异常这个特性带来的损耗吗？你的性能瓶颈真的是异常导致的吗？)。当然还有一个原因会导致我们放弃使用异常，那就是学习成本，对于很多C++新人来说，使用异常需要注意的事情太多，很多新人容易误用，从而引起更多的问题。我想这也是google在其C++代码规范中禁止使用异常的原因吧(尤其是异常缺点的介绍，感觉就是太复杂，我不想处理，所以我不用异常)。相对于C++，C#的异常处理会更简单些，因为它有垃圾回收等机制的保障，这大大提升了程序员的生产力，而且C#的整个体系架构就是基于异常的，如果我们不使用异常，那么了我们如何处理那些基础库抛出的异常？

# 2. C++异常需要注意

因为C++异常使用对于开发要求更高，所以我们一起重点看看C++的异常处理应该注意些什么？

**1. 异常对象就是普通对象：** 也许你已经注意到，我们前面例子中的异常类并没有继承STL中的std::exception。因为C++并没有强制要求自定义异常类需要继承某个基类，只要抛出的异常类型与catch语句中的异常类型匹配，那么异常就能被捕获。尽管如此我们依然希望在应用程序开发过程中C++也最好继承STL的std::exception(CRL其实也没有具体限制异常的类型，但是在C#2.0以后，C#要求自定义异常必须继承于System.Exception)。因为有时我们会因疏忽而忘记在应该捕获异常的地方处理异常，如果我们自定义的异常继承于std::exception, 那么至少它会被std::exception的catch语句所捕获，并被记录处理(当然这只是一些应该捕获异常的场景，并不是任何异常我们都要一个不漏的将其捕获，下面我们会有进一步介绍)。

**2. 构造函数中抛出异常：** 异常处理是一个比较棘手的问题，异常抛出的代码并不知道调用方所处的上下文，所以一般异常都由调用代码捕获并处理。但对于C++来说，其中一个比较大的问题就是资源管理。一般情况我们会手动的在catch语句中销毁之前在堆上分配的内存或资源(文件句柄，数据库连接等)。C#的final语句大大减少了重复回收资源的代码。但是构造函数也能那么处理异常吗？请看下面代码:

```C++
class Child1;
class Child2;
class Parent
{
public:
    Parent(){
        m_pChild1 = new Child1();
        throw ParentCntorException();
        m_pChild2 = new Child2();
    }
    ...
private:
    Child1* m_pChild1;
    Child2* m_pChild2;
}

int main(){
    Parent* pParent;
    try{
        pParent = new Parent();
    }
    catch(ParentCntorException& ex){
        delete pParent; //这样做真的可以解决内存回收吗？
    }
    ....
}
```

例子中m_pChild1分配的内存并没有被回收。因为异常在构造函数中抛出，Parent对象并没有完全构造出来，C++编译器并不会返回这个对象，pParent依然是空指针。所以catch中的delete并不会调用Parent的析构函数。那么应该如何正确处理构造函数的异常呢？下面是正确的打开方式:

```C++
class Parent{
public:
    Parent(){
        try{
            m_pChild1 = new Child1();
            throw ParentCntorException();
            m_pChild2 = new Child2();
        }
        catch(ParentCntorExcepion& ex){
            delete m_pChild1;
            throw; //继续抛出异常，通知调用端
        }
    }
}

int main(){
    Parent* pParent;
    try{
        pParent = new Parent();
        ...
    }
    catch(ParentCntorException& ex){
        delete pParent; //这里其实就是nullptr, 不delete也可以
        // 记录异常
    }
}
```

因为我们无法在Parent外面正确处理已经分配的内存，所以我们需要在构造函数中捕获异常，并做相应的资源回收，然后继续常抛出异常提醒调用代码(我们当然可以通过判断构造函数返回是否为空，来减少异常抛出所带来的性能损耗。但构造函数返回失败的可能性是非常的小，如果每次调用构造函数我们都要先判断是否为空，那么这在代码可读性以及性能等考量上也未必是最优的)。上面的代码是不是看着很繁琐？其实处理资源泄露的最简单的方法就是使用RAII(Resource Acquisition Is Initialization)，unique_ptr、lock_guard就是RAII非常好的应用，它们利用对象出作用域后，自动调用析构函数来实现。将m_pChild1和m_pChild2封装在unique_ptr中，Parent的构造函数就不再需要捕获异常，清理内存的操作了(C#因为有垃圾回收器，所以完全不需要考虑构造函数抛异常的情况)。

**3. 析构函数不能抛出异常：** 构造函数是一个坑，析构函数更是如此，因为它不仅会让你内存泄漏，甚至会让你程序崩溃。请看下面例子：

```C++
class Parent{
public:
    ~Parent(){
        delete m_pChild1;
        throw ParentDetorException();
        delete m_pChild2;
    }

    throwException(){
        throw std::exception();
    }
}

int main(){
    try
    {
        Parent p;
        p.throwException();
    }
    catch(...){

    }
}
```

上面例子的结果就是程序崩溃了。很简单，在调用throwException函数异常抛出后，编译器会调用Parent的析构函数析构Parent对象，但是析构函数又抛出了一个异常，连续两个异常抛出，程序唯一的应对方法就是崩溃。所以**在析构函数里面不允许抛出异常**。下面是析构函数应该处理的操作：

```C++
class Parent{
public:
    ~Parent(){
        try{
            delete m_pChild1;
            throw ParentDetorException();
            delete m_pChild2;
        }
        catch(ParentDetorException& ex){
            delete m_pChild2;
        }
    }
}
```

虽然C#中我们一般不会显示的实现析构函数，但是如果你为了优化性能手动实现析构函数，那么你也必须保证析构函数不应该抛出异常。

**4. 用引用捕获异常：** C++和C#在抛出异常有很大的区别，C#的异常都是创建在堆上，因为有垃圾回收器，我们不需要担心异常对象泄漏的问题。但是C++就不同了，它的异常不仅可以创建在堆上还可以创建在栈上甚至作为静态对象保存在全局区域，所以在异常捕获的地方(catch语句)，如果我们用delete贸然删除一个异常指针，很可能发生不可以预知的后果。所以我们推荐使用捕获异常的引用，因为它比指针异常多一次拷贝，但是又比值异常少一次拷贝(值异常抛出前会再做一次拷贝，延长异常对象的生命周期，不再退出try语句后被析构)，是一个折中的方案。另外在catch语句中捕获的引用异常，在其再次被抛出时，我们可以保证异常对象的完整性，在捕获有继承关系的值异常中再次抛出异常，异常对象会被downcasting成基类对象，比如下面的例子:

```C++
class CustomException : public std::exception{};
void reThrowException(){
    try{
        throw CustomException();
    }
    catch(std::exception ex){
        // 处理异常，再次抛出异常
        throw;
    }
}

int main(){
    try{
        reThrowException();
    }
    catch(CustomException ex){
        // 并不会捕获异常
    }
    ...
}
```

**5. 捕获线程异常：** 异常的复杂还表现在多线程的处理上，如果一个主线程要捕获一个次线程抛出的异常，那是不可能的。因为异常对象被保存在每一个线程的本地存储中(TLS)。当然如果真要传递这个异常，也不是没有办法。因为前面我们说了，异常就是一个普通的对象，既然我们一般堆栈的对象都可以传入传出一个线程，那么只要将一个线程中的异常对象通过地址或者全局对象传递给另一个线程不就可以了。C++11引入了exception_ptr对象，我们可以将其用于异常传递，但是这样的方法看上去不是很自然(聊胜于无)。C#提供的Task很好的保持了异常使用的一致性。我们可以直接在一个线程中利用try-catch直接捕获另一个线程中的异常，当然这个异常并不是那个真正抛出的原始异常，而是TPL对真正异常封装后的一个聚合异常(AggregateException)，但是我们可以通过InnerExceptions访问具体的异常。

```C#
static void ThrowIndexOutOfRangeEx()
{
    throw new IndexOutOfRangeException();
}

static void Test()
{
    Task<string[]> task1 = Task<string[]>.Factory.StartNew(() => ThrowIndexOutOfRangeEx());

    try {
        task1.Wait();
    }
    catch (AggregateException ae) {
        ae.Handle((x) =>
        {
            if (x is IndexOutOfRangeException)
            {
                // 处理对应的异常
            }
        });
        //var exceptions = ae.InnerExceptions
    }
}
```

以上就是C++处理异常时应该注意的问题。

> Note：C++中还有一个叫异常规格的概念，就是在函数后面加一个throw()修饰，括号里面可以指定异常类型，但是C++11已经将这个概念摒弃，取而代之的是用noexcept，noexecpet是修饰函数是否应该抛出异常，比如像析构函数，swap以及move构造等函数是天生符合noexcept的，有了noexcept编译会对其代码进行优化，从而减少调用堆栈的展开。

# 3. 异常的使用原则

了解了C++异常使用细节，我们一起总结下异常使用的一个指导原则。

1. 在代码可读性，复杂性以及易用性没有改变的情况下尽量不要使用异常或抛出异常；
2. 如果我们可以通过预先的处理避免一些异常的发生，我们就应该做预处理(比如提前判断状态的有效，在调用具体的方法)。
3. 如果我们要处理异常，我们需要尽可能的保证代码异常安全。异常安全一般从4个层次保证：(1)无异常抛出，就像之前提到的析构函数，swap，move都应该定义为无异常函数。(2)强异常安全，这里我们需要保证异常发生后，调用前后状态保持一致性(比如数据的事务操作)。(3)基本异常安全，这就是我们上面提到的保证资源不泄漏。(4)没有异常安全保障，这就不用解释了，就是异常发生，资源泄露，对象状态被破坏。
4. 在不能保证强异常安全的情况下，我们因该抛出异常让持续崩溃。因为不是所有情况我们都可以还原之前的状态，与其捕获异常，并用复杂的代码处理不正确的状态，还不如让程序早点退出。
5. 在合适的地方处理或抛出自定义异常，而不是通用异常类(System.Exception或std::exception)。这种省力的做法，会存在很多隐患，因为每个独立的异常类都代表着不同的异常，而每种异常处理的方法是不一样的，如果都用基类处理，势必会遗漏很多重要的信息。
6. 不加区分的用基类捕获所有异常，对调用代码来说是一个灾难。因为客户代码才是最清楚那个调用时间点，如何处理对应的异常。如果异常都被函数捕获并淹没，而且代码状态又不能恢复到原始的状态，这很可能让调用代码无意识的使用一个已经出了错误的对象，这样的后果是非常严重的。
7. 自定义在接口中抛出的异常，因为我们可能不经意中暴露服务或基础库内部的异常给用户，而这些异常又不应该给用户知道的。
8. 按继承顺序捕获异常。如果异常有继承关系，那么我们最好先捕获具体的异常，然后获基类异常。在C++中如果顺序错误，很可能具体异常被基类异常给捕获，导致catch具体异常的语句永远不会被执行到。C#不需要考虑这个问题，因为编译代码时，编译器就会报错提醒。

# 后记

异常的讨论一直存在，异常的使用是需要物理成本的，但是开发效率也是有成本的，这之间的关系，不就是应该我们自己做出权衡(而不是因为学习曲线高而放弃使用它)，世界上本来就没有绝对的对与错，只有这个方案对于当前问题是不是局部最优。
