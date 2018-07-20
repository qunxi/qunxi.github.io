---
author: qunxi
create: 2018-07-15 18:43+08:00
update: 2018-07-19 18:43+08:00
layout: page
title: "C++11之右值与右值引用(一)"
description: ""
comments : true
categories:
tags:
- 语言
- 技术
---

# 前言

自从C+11的出现，C++语言推出了很多新的特性，这些特性不仅提升了C++开发效率，而且还对C++98/03进行了一些语义的扩展。尤其是右值引用这个概念，在C++11中被大量使用，所以对右值和右值引用的正确理解是用好C++11的基础。今天我们就一起聊聊什么是右值和右值引用。
<!--more-->

# 右值和右值引用(&&)

要了解右值引用就必须先了解什么是右值。最开始的定义就是以赋值符(=)为界，等号右边为右值，反之为左值。这种定义虽然直观，但不全面。比如下面的代码:

```C++
int a = 10; // a为左值，10为右值
T foo(X x){
    // x 为左值，但是在调用时会通过拷贝构造一个临时变量，这个临时变量为右值
    // to do something
    return T();
}
a + b;  // a和b是一个左值，但是整个表达式是右值
X x; // x 为左值
foo(x); // 虽然没有赋值符，但是foo(x)的返回值是一个右值。另外这里编译器会通过X的拷贝构造函数生成一个临时变量作为foo的入参
```

首先C++中任何表达式要么是左值要么是右值，所以光靠相对赋值符的位置判断左右值并不科学，尤其foo函数的入参x或其返回值，编译器会通过拷贝构造函数生成一个临时变量，而这个临时变量是一个右值（右值通常是一个没有命名的临时变量）。一个更合理判断左右值的方法是对表达式(或对象)取地址，如果可以获取地址的则为左值，否则为右值。参看下面代码如下：

```C++
int a = 10;
&a; //左值
&10; // 编译错误，右值
&foo(); // 编译错误，右值
&string("hello world"); // 编译错误，右值
&(a + b);  // 编译错误，右值
```

**为什么右值不能被取地址？** - 因为临时变量的生命周期都很短而且其内存销毁并不完全由用户控制，所以通过地址访问的内容很可能在使用前就已在编译器背后被销毁，这是非常危险的行为，编译器不用允许这种错误发生。所以通过能否取地址判断左右值更加准确。

引用和指针类似，它是一个已有对象的别名，很明显，如果一个非常量左值引用(T&)也可以引用右值的话，这同样是一个非常危险的行为，所有这些错误必须在编译时被发现并告知用户，所以**指针**和**非常量的左值引用**是不允许绑定右值的。当然我们可以通过拷贝构造完成值的传递，但是这明显摈弃了指针和引用在性能以及灵活性的优势。比如下面的代码：

```C++
class A{
    public:
        A()
            m_pChar = new char[1000];
        }
        ~A(){
            delete []m_pChar;
        }
    private:
        char* m_pChar;
};

A funcA(){
    return A();
}

void funcA1(A a){
}

void funcA2(A& a){
}
```

排除编译器的优化funcA2的性能会高于funcA1，道理很简单funcA1会调用A的拷贝构造函数生成一个临时变量，而且函数结束后还要销毁临时变量分配的内存。funcA2虽然性能更高，但它不能绑定右值，比如`funcA2(funcA())`这样的调用，编译器是不能通过的。也许你已经注意到**非常量**这个修饰，其实C++98是允许右值绑定到**常量左值引用(const T&)**，原因很简单，我们不可能通过常量左值引用来修改右值(临时变量)，这就不会有什么风险。说道这里，也许你会问，那么C++11引入右值引用(T&&)有什么用？前面不是说修改临时变量是一个很危险的事吗？是的，如果右值引用和左值引用一样，仅仅是一个对象的别名，那么引入右值引用的确像前面说的一样有很大的风险存在，但是C++11右值的引用其实不仅仅是别名的作用，它更强调的移动(Move)语义和完美转移(Perfect Forwarding)。

> Note：如果你在你的编译器上编译过之前的代码&string("hello world")和&foo()，也许你看到的结果并不像我之前描述的，你可以取到右值的地址！！不要惊慌，请务必确认你的编译器已经关闭了语言扩展(\Ze)

# 移动语义(Move Semantics)和完美转移(Perfect Forwarding)

## 移动语义&拷贝语义

* 拷贝(Copy)语义：说到Move语义，我们不得不提一提**Copy语义**。C++11之前我们可以通过拷贝构造或者赋值操作符对一个对象进行拷贝。如果我们不显示的定义这两个函数，编译器会自动生成一个默认的拷贝构造函数和赋值操作函数。默认情况下拷贝构造进行的是浅拷贝，它会隐含潜在的风险，如果其成员指针同时指向一块内存地址，那么任何一个对象对内存进行了销毁都会影响另一个拷贝的对象，从语义上讲，从对象进行拷贝的那一刻之后，拷贝对象和原始对象应该是独立互不影响的。所以一般情况下，如果有内存的管理，我们都会显示定义拷贝构造和赋值操作，并实现深拷贝。所有的方法都有两面性，深拷贝带来是性能的问题，我们需要申请内存，并依次拷贝原始对象的内容到拷贝对象中，如果拷贝对象生命周期结束，我们还需要销毁之前分配的内存。

* 移动(Move)语义：简单的说，它就是把一个对象（临时变量）的所有权从原来的对象转移到另一个对象上，原来那个对象（临时对象）不再拥有对象控制权。现在回到第一节留下的问题，为什么说右值引用不会带来风险，因为右值引用引入了Move语义，有了Move语义，只要右值引用了某个临时变量（右值），那么临时变量的控制权就完全转移到了用户手上，我们就再不担心临时变量在我们不知情的情况下默默的被销毁，而且移动语义的性能和浅拷贝的性能是完全一样的，它唯一要做的就是将原始对象的指针设为空。

下面看一个拷贝和移动语义的实现：

```C++
class MyString{
    public:
        ...
        MyString(const MyString& lhs)
            : data_(new char[lhs.size_])
            , size_(lhs.size_)
        {
            memcpy(data_, lhs.data_, lhs.size_);
        }

        MyString& operator=(const MyString& lhs){
            if(this != &lhs){
                data_ = new char[lhs.size_];
                size_ = lhs.size_;
                memcpy(data_, lhs.data_, lhs.size_);
            }
            return *this;
        }

        MyString(MyString&& rhs) //注意虽然这是一个move构造，但是这里rhs是个左值
            : data_(rhs.data)
            , size_(rhs.size)
        {
            rhs.data_ = nullptr;
            rhs.size_ = 0;
        }

        MyString& operator=(MyString&& rhs){
            data_ = rhs.data_;
            size_ = rhs.size_;
            rhs.data_ = nullptr;
            rhs.size_ = 0;
            return *this;
        }
        ...
    private:
        char* data_;
        int size_;
}
```

很明显移动构造和赋值比拷贝的构造赋值效率高很多，这里右值引用并不是const的，因为我们需要清空原始右值的状态来实现转移语义。使用移动语义的场景非常多尤其在C++11以后，最典型的一个例子就是uinque_ptr，在C++11之前因为没有实现移动语义，所以auto_ptr是通过拷贝构造来实现所有权转移的，这会带来语义的矛盾，拷贝语义被移动语义覆盖，对于客户代码来说会很困惑，明明是拷贝为什么我的原始对象不见了。所以在C++11之前是不建议在容器内保存auto_ptr的。C++11之后，标准库引入了unique_ptr，因为它更加的安全，所以auto_ptr已经不建议使用了。

```C++
vector<auto_ptr<int>> arrOrgins;
arrOrgins.push_back(auto_ptr<int>(new int()));
arrOrgins.push_back(auto_ptr<int>(new int()));
arrOrgins.size(); // 大小为2
auto arrCopy = arrOrigins;
arrOrigins.size(); // 到校2且arrOrgins里的指针为空，arrOrgins很容易被误用
vector<unique_ptr<int>> arrOriginUptrs;
arrOriginUPtrs.push_back(make_unique<int>());
arrOriginUPtrs.push_back(make_unique<int>());
auto temp = arrOriginUPtrs; // 编译器报错，避免出错
auto arrPUs1 = move(arrOriginUPtrs); // 用户显示的转移
arrOriginUPtrs.size(); // 数组大小为0
```

这里我们使用了一个标准库的move函数，它显示的告诉用户调用这个函数是一个移动操作，但实际上真正的实现只是将这个左值类型强转成右值，后面章节我们会介绍`move`的具体实现。

## 完美转移

右值引用不仅仅带来移动语义操作，而且解决了泛型编程中完美转移的问题。下面我们来看看什么完美转移，

```C++
// part1
// template<typename T1, typename T2> tuple<T1, T2> make_tuple(T1 t1, T2 t2)
// {
//    return std::tuple<T1, T2>(t1, t2);
// }

// part2
template<typename T1, typename T2> tuple<T1, T2> make_tuple(T1& t1, T2& t2)
{
    // ...
    return std::tuple<T1, T2>(t1, t2);
}
// part3
template<typename T1, typename T2> tuple<T1, T2> make_tuple(const T1& t1, const T2& t2){
    // ...
    return tuple(t1, t2);
}
int a = 10;
float b = 20.0f;
make_tuple(a, b); // #1 part2
make_tuple(10, 20.0f); // #2 part3

```

上面代码中(#1和#2)虽然可以匹配到part1，但是它是通过临时变量传入参数的，对于基本类型还可以接受，如果是其他用户定义的对象就会影响效率（拷贝构造）。使用part2和part3的函数重载可以完成左右值的引用传参，但是并不完美，如果我们要调用`make_tuple(a, 20.0f)`虽然它可以映射到part3，但是很明显如果要更完美的匹配，我们应该再重载一个像`make_tuple(T1&, const T2&)`这样的一个函数。如果所有参数都要完美匹配的话make_tuple必须实现4种函数签名。当然我们知道tuple是可以支持更多参数的，如果有n个参数，那么我们就需要实现2^n个函数，这样是一个很庞大的工作量。另外如果传入的参数是右值，那么最终会匹配到part3，而part3的实现会失去右值的移动语义，如果你希望泛型也能充分利用移动语义的优势，那么C++11之前是无能为力的。如何让T1， T2自动的推断出其入参是左值还是右值，并保留做右值的特性，这就是完美转移所需要完成的能力。

C++11要实现完美转移必须具备两个概念，一个是**通用引用(universal reference)**，另一个是**引用折叠(reference collapsing)**。

> **通用引用(universal reference)**是一个既可以引用右值，又可以引用左值的引用。必须要注意，它和右值一样也是`&&`来表示。

那么右值引用和通用引用有什么区别呢？什么情况`&&`代表的是右值引用，什么时候代表的是通用引用呢？请参考下面的代码：

```C++
int a = 1;
auto&& ura = a; //通用引用
int&& ra = 1;   //右值引用
template<typename T> fun(T&& t); //通用引用

void vector<T>::push_back(T&& t); //右值引用
void vector<T>::emplace_back(Arg&& t); //通用引用
```

也许你已经发现了其中的秘密，`&&`修饰的类型需要通过类型推导的，那么它就是通用引用，否则为右值引用。因为调用push_back时，T的类型已经确定，所以`push_back`是右值引用，而`emplace_back`的Arg类型需要推导，所以是通用引用。

另一个概念是引用折叠(reference collapsing)，以下是推导公式

```C++
A& & -> A&
A& && -> A&
A&& & -> A&
A&& && -> A&&
```

有了引用折叠和通用引用，我们只需要实现一个函数就可以替代原来的2^n个重载函数，而且还保留了完美的移动语义。代码实现如下：

```C++
template<typename T1, typename T2> tuple<T1, T2> make_tuple(T1&& t1, T2&& t2)
{
    // ...
    return std::tuple<T1, T2>(std::forward<T1>(t1), std::forward<T2>(t2));
}
int a = 10;
float b = 10.0f;
// make_tuple(int&& &, int&&) -> make_tuple(int&, int&)
make_tuple(a, b);
// make_tuple(int&& &&, int&& &&) -> make_tuple(int&&, int&&)
make_tuple(1, 1.0f);
// make_tuple(int&& &&, int&& &) -> make_tuple(int&&, int&)
make_tuple(1, b);
// make_tuple(int&& &, int&& &&) -> make_tuple(int&, int&&)
make_tuple(a, 1.0f);
```

是不是代码实现简单了很多，这里我们使用了forward函数，它就是将左值或者右值原封不动的传递给后面的函数实现完美转移。forward的具体实现我们会在下节介绍。

# 后记

到这里右值和右值引用的基本概念介绍的差不多了，由于篇幅的限制，我会在下一篇文章进一步介绍这章节留下的一些问题。比如`move`函数和`forward`函数的实现，以及右值引用的一些应用和误解。