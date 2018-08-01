---
author: qunxi
create: 2018-07-21 18:43+08:00
update: 2018-07-31 18:43+08:00
layout: page
title: "C++11之右值与右值引用(二)"
description: ""
comments : true
categories:
tags:
- 基础
---

# 前言

[上一篇文章](https://qunxi.github.io/2018/07/15/c-plus-plus-rvalue-reference-session-1.html)我们介绍了右值和右值引用，以及C++11引入的移动语义和完美转移等概念。本文我们将继续讨论上一篇文章遗留的问题以及关于右值引用的一些具体应用实现。
<!--more-->

# move和forward的实现

上一篇我们讨论了移动语义和完美转移的基本概念，而且也提到了在C++标准库(STL)里面实现相应的辅助函数**std::move**和**std::forward**，下面我们一起来看看这两个函数的具体实现和作用。

**std::move**函数名的表面意思是将一个对象转移(move)到另一个对象，但事实并非如此，move背后的实现仅仅是将move的入参强制转换成右值。代码如下：

```C++
template<typename T>
using remove_reference_t = typename remove_reference<T>::type;

template<typename T>
remove_reference_t<T>&& move(T&& param){
    return static_cast<remove_reference_t<T>&&>(param)
}

move(1);//1为右值，推导出move(T param)->static_cast<T&&>(param)
int i = 2;
move(i);//i为左值，推导出move(T& param)->static_cast<T&&>(param)

```

模板入参是一个Forwarding Reference，在函数中我们直接用static_cast将其转换成右值。因为右值具有移动语义，所以通过move函数转换后，返回值就具备了移动构造和移动赋值的能力。当然这里必须声明，被移动的对象必须实现移动拷贝和移动赋值，而且move的对象不可以是一个常量。比如下面的实现:

```C++
const MyString str1("const string");
MyString str2 = move(str1); //这里不会调用移动构造
```

因为const MyString经move转化后是一个const右值，除非你实现移动构造的入参声明为const MyString&&，否则编译器会调用被移动对象的拷贝构造函数进行拷贝操作。如果你实现了以const MyString&&为参数的移动构造，那么这个移动构造又违背了移动语义，因为你无法转移原始的对象的所有权（设置原始对象为空）。

> move函数并不一定能真正的调用移动构造实现移动语义，除非满足了上面提到的两点

**std::forward**函数是将一个左值或右值对象，原封不动的传递到下面的函数。forward和move函数一样其内部的实现也是做了类型转化，唯一的区别是，forward的是有条件的转化右值（通过引用折叠的规则推导出最终的类型），forward表达的是一种关于值类型（左值或右值）不变的传递，而move是无条件的进行右值转换。代码如下：

```C++

template<typename T>
using remove_reference_t = typename remove_reference<T>::type;
template<typename T>
using is_lvalue_reference_v = typename is_lvalue_reference<T>::value;

template<class T>
T&& forward(remove_reference_t& Arg) noexcept
{
    return (static_cast<T&&>(Arg));
}

template<class T>
&& forward(remove_reference_t&& Arg) noexcept
{
    static_assert(!is_lvalue_reference_v<T>, "bad forward call");
    return (static_cast<T&&>(_Arg));
}
```

上面是forward的两种重载函数，一个适配左值一个适配右值。你也许有疑问，为什么不直接用Rorwarding Reference实现forward函数呢？像下面的代码：

```C++
template<class T>
T&& my_forward(T&& a){
    return static_cast<T&&>(a);
}

int a = 1;
my_forward(a); //#1 返回左值 static_cast<T& &&>(a)
my_forward(1); //#2 返回右值 static_cast<T &&>(a)

std::forward<int>(a); //调用第1个左值的重载函数，返回左值
std::forward<int>(1); //调用第2个右值的重载函数，返回右值
```

如果forward只有上面这中使用场景，那么我们完全没有必要像标准库那样繁琐的实现，但事实是这样的吗，我们往下看:

```C++
template<class T>
void func(T&& a){
    func2(my_forward(a)); //#3 a是一个左值，func->func2的参数类型已经改变
    func2(std::forward<T>(a)); //调用第1一个左值的重载函数，返回右值
}

template<class T1>
void func2(T1&& b){
    // do something
}

func(1); // T是一个右值类型，但是a是左值，所以my_forward返回的是左值（非完美转移），而std::forward返回的是右值（完美转移）

```

> 注：Forwarding Reference的推导，如果T是一个左值那么它的推导为T&（左值引用），而T如果是一个右值，它推导为T（**右值**而不是右值引用）。参看#1和#2。

场景#3的调用暴露了问题，func传入右值，forward出来后是一个左值，这违背了完美转移的定义。所以直接用Forwarding Reference实现forward函数是不能完全覆盖所有使用场景的。

# 关于右值引用的一些思考

C++11引入右值听上去很是美好，它可以提高效率，可以实现完美转移和移动语义。那么事情的背后真的是这样吗，我们可以免费的获得这些特性吗？我们一起思考下下面的问题。

## 1. 右值可以大大提高性能？

答案当然是否定的。首先右值的性能取决于对象移动构造和移动赋值的实现方式，如果某种类型它的move语义实现没有一个相对拷贝来说好的性能实现，那么利用右值并不能带来任何性能上的提升，比如标准库中的string，如果它的拷贝构造是按照CopyOnWrite的方式实现的，那么在一开始拷贝或复制操作的效率并不比移动构造的性能低（现在标准库基本不使用COW实现，即便如此，在处理一个小字符串的时候，string的移动构造也不见得比拷贝构造快，因为string实现了SSO[Small String Optimization]）。其次即使你实现了一个相对拷贝构造而言更高效的移动构造和移动赋值，你能保证你的实现一定比编译器优化还高效吗？比如RVO(Return Value Optimization)，参看下面的例子:

```C++
MyString func(){
    MyString a;
    ....
    return a; //#1 不会调用移动构造
    return move(a); //#2 调用移动构造
}

MyString str = func();
```

理论上func返回的是右值，那么理论上#2的move操作应该比#1的copy操作性能更高。但是这里的局部变量a满足编译器的RVO优化，所以str直接引用了局部临时变量a的地址空间，并不需要调用拷贝构造。

> **RVO**必须满足以下两个条件：1. 局部变量的类型和函数返回类型一致；2. 局部变量就是返回值。

当然这些都是在一定的条件下我们做的假设，如果我们确定move构造确实能给我们带来效率提升，那么我们也必须选择右值。

## 2. 编译器会为我们生存默认的Move操作吗？

在C++11之前，如果我们没有定义拷贝构造或则拷贝赋值操作，编译器会自动为我们生成一对隐式构造和拷贝，这大大方便我们手工编写一些意义不大的拷贝操作。C++11之后是否也会为我们生成相对应的移动操作呢（移动构造和移动赋值）？答案是-不一定。比如下面的代码：

```C++
class A{
    public:
    A(){
    }
    A(const A&){
    }
    A& operator=(const A&){
    }
}
A a = A();
A b = move(a); // 调用的是拷贝构造
```

因为对象A并没有显示的定义移动操作，所以A b = move(a)依然调用的是拷贝构造函数，即使我们这里强制对a做了右值转换。很明显如果强制给代码隐式生成移动操作，会对C++11之前的代码升级带来隐患，因为如果C++11之前的代码，用户已经重新实现了默认的拷贝构造或拷贝赋值亦或是析构函数，那么说明编译器自动生成的这些函数所实现的拷贝已经不适合用户的需求，那么默认生成的移动操作也会有潜在的问题。所以在C++11规定除非代码没有显示的声明拷贝构造，拷贝复制，移动构造，移动赋值和析构函数中的任何一个，那么编译器才会自动生成移动操作，否则不会生成移动构造或者移动赋值。反过来，如果你自定义了这五个函数中的其中任何一个，那么说明你必须实现其他四个，否则就会对代码带来隐藏的风险， 这就是五项原则(Rules Of Five)。

## 3. 为什么不建议重载或特化Forwarding Reference？

这个问题其实很容易回答，首先Forwading Reference出现的原因之一就是为了解决过多参数带来函数重载的不便；其次如果特化了模板参数，很容易导致因为一些类型特化的遗漏，从而导致再次匹配到Forwarding Reference的重载函数。比如下面的例子：

```C++11
class A {
public:  
    template<typename T>
    explicit A(T&& n){
        name_ = forward<string>(n);
    }
    A(const A&){
        ..
    }
private:
    string name_;
}

A a("xyz");
A a2(a); //调用的不是拷贝构造，惊讶不？

```

上面的代码永远都不会调用它的拷贝函数，而且最终导致编译错误，因为a是一个非常量的左值，所以在函数匹配时，它更倾向于匹配第一个模板函数，即使拷贝函数不是模板函数，所以对Forwarding Reference函数重载要千万的小心。

## 4. 为什么建议使用emplace函数？

答案是它可以大大提升你的性能。我们可以通过完美转移将参数传递到真正需要它的地方再进行处理，而不是经过几次中间过程的隐式转换导致性能下降。STL标准库中很多容器都实现了类似的函数。下面我们以vector容器为例：

```C++
vector<string> vs;
vs.push_back("abcde");
vs.emplace_back("abcde");
//push_back和emplace_back的定义
//void vector<T>::push_back(T&& t);
//void vector<T>::emplace_back(Arg&& t);
```

虽然上面push_back和emplace_back的使用看上去完全一样，但是调用背后的逻辑却完全不同。push_back调用时因为入参不匹配(char[]->string)，会生成一个类似string("abcde")的临时变量，这里会调用一次拷贝构造。接着push_back会将这个临时变量拷贝到vs的成员中，这里调用了一次移动构造（因为右值）。而emplace_back则完全不同，它直接将char[]作为参数直接传递到vector中构造，免去了临时变量的构造，如果我们做大量数据的插入，性能将会有很大的提升。

# 后记

为什我要写这篇文章，因为我发现其实很多人说自己了解C++11，但是一旦问到细节，很多人只是停留在用过或知道的阶段，包括我在写这两篇文章的过程中，我发现自己也在不停纠正自己之前错误的认知（通过更深入的查阅了一些资料）。“会用”不代表真的知道，知道不代表真的**会用**。
