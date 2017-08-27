---
author: qunxi
create: 2017-07-20 18:43+08:00
update: 2017-07-20 18:43+08:00
layout: page
title: "设计模式-Bridge模式"
description: ""
comments : true
categories:
tags:
- 设计
- 模式
---
# 前言

封装是代码设计非常重要的关注点，在编写代码时我们到底应该封装什么，怎么封装？希望通过这篇文章关于Bridge模式的介绍能给你一点点启示。
<!--more-->

# 1. 抽象和实现的分离

在介绍[代码设计](https://qunxi.github.io/2017/01/30/object-oriented-design.html)时，我们有提到要**面向接口编程，而不应该面向实现**，因为细节容易引入更多变化和依赖。所以通常我们会抽象共性作为接口，然后利用继承方式实现细节。比如我们要实现一款支持pdf、txt、epub的阅读App，我们最先考虑的是抽象出三种文本格式的解析和渲染等共性接口，以及三种文本的具体实现。

```
interface IReader{
    void Open(string path);
    void Close();
    string GetBookName();
    TableOfContent GetTableOfContent();
    Content GetContentOfChapter(int chapter);
}

class PdfReader: IReader{
}

class EpubReader: IReader{
}

class TxtReader: IReader{
}
```

理论上作为App其他部分(客户代码)应该只依赖于`IReader`接口，其它三个子类对它来说应该是完全透明的。这样具体实现不论发生什么改变，都不会影响到客户代码。这里你可能会提出疑问，如果客户代码完全不依赖于具体实现，那么这些具体地对象在什么地方构造呢？是的，任何相关对象是不可能完全去除依赖，但是我们利用[工厂模式](https://qunxi.github.io/2017/02/18/design-pattern-factory.html)或者IoC容器来封装构建的细节，这样我们不至于将来因为构造的具体对象或细节发生变化，而影响到客户代码。

如果客户代码和文本解析的关系是相对稳定，那么利用继承来完成抽象与实现的分离就是最理想的方法。但是有些时候变化是相互影响的，比如接口虽然相对于客户代码说是稳定的，但是也有改变的可能性，这时一旦接口变了，那么客户代码和接口的实现类都会受到影响，如何避免接口部分的变动或是实现部分的变化导致的代码影响最小呢？靠继承很明显无法实现这点，因为继承将抽象和实现固定的绑定在了一起，这注定会导致变化的传导性。所以我们需要Bridge模式要解决以上问题。

# 2. Bridge模式

如何将抽象与实现完全的分离？这里我们必须去除抽象和实现的直接依关系-继承。如果从下往上提取抽象，我们很难将抽象和底层实现分开，而且这样的抽象容易设计出自以为是的接口，并且一旦底层方式变化，上层代码都会发生变化。好的抽象应该来自上层(客户代码)，只有真正的客户才知道他们需要的是什么，这就是**依赖倒置原则**。下面我们就来重构之前那个阅读APP。

首先从客户代码角度，我们希望对于所有格式的电子书都有统一操作电子书的抽象行为，比如打开、获取目录、获取章节内容等，所以这里的抽象部分我们可以定义如下：

```

interface IReader{
    void Open(string path);
    void Close();
    string GetBookName();
    TableOfContent GetTableOfContent();
    Content GetContentOfChapter(int chapter);
}

class Reader : IReader{

    private IReaderImpl readImpl;

    public void Open(string path){
        readImpl.Open(path);
    }

    public void Close(){
        readImpl.Close();
    }

    public string GetBookName(){
        return readImpl.GetBookMetaData().Name;
    }

    public TableOfContent GetTableOfContent(){
        return new TableOfConent(readImpl.GetBookMetaData());
    }

    public Content GetContentOfChapter(int chapter){
        return new Content(readImpl.GetBookContent.Chapters[chapter]);
    }
}
```
`IReader`是客户代码制定的抽象部分，作为客户代码它只需要知道`IReader`，至于`IReader`背后的实现对于客户代码完全不需要了解。同时作为底层电子书解析库`IReaderImph`，它也完全不需要依赖客户代码定义的`IReader`接口。下面我们再来看看实现部分的代码：

```
interface IReaderImpl{
    void Open(string path);
    void Close();
    BookMetaData GetBookMetaData();
    BookContent GetBookContent(); 
    ...
}

class PdfReaderImpl : IReaderImph{

}

class TxtReaderImpl: IReaderImpl{

}

class EpubReaderImpl: IReaderImpl{

}

```
上面的实现代码已经完全的与客户定义的抽象代码独立开来，无论是哪边发生了变化都不会影响另外一部分，这就是Bridge模式。下面来看看Bridge模式的完整定义和类图：
> 将抽象部分与它的实现部分分离，使它们都可以独立地变化。

![bridge pattern](/post-images/2017_7_20_bridge_pattern.png)

Bridge模式就是利用组合的方式将分离的抽象和实现桥接在一起，使其不像继承那样强耦合，并且拥有了组合的灵活性。当然这里唯一的代价就是要额外的添加一个类的层次结构。

# 3. Bridge模式的思考

1. Bridge模式不仅仅可以帮组我们隔离变化，而且还能很好的做到细节的封装。比如C++中通过共享的引用计数器来管理内存的智能指针`shared_ptr`，也是一个简化后的Bridge模式。整个计数器的实现是`shared_ptr`的最重要的算法，但是作为客户代码我们机会觉察不到它的存在，这就是Bridge模式对细节封装的强大之处。

2. 也许有人会问既然Bridge模式能够如此强大，那么我们为什么不都用Bridge模式来写所有的代码？这里我必须指出**凡是**这两个字永远不应该出现在设计中，设计是有一定的结构和套路，但是设计的精髓更重要的是现实和理论的取舍。如果有一种的万能解决的方法，那么我们又何必再学习。就像Bridge模式在带moshi来好处的同时也增加了编码复杂度，过度设计不仅是成本的浪费，也是一种惰性思考的借口。

3. Bridge模式和[Adapter模式](https://qunxi.github.io/2017/03/25/design-pattern-adapter.html)在实现上有些相识之处-它们都是用组合的方式来实现的，但是Adapter模式更强调的是对已有实现的重用和接口的适配，而Bridge模式更强调的是实现和抽象的分离，这一般是在设计之初需要非常认真考虑的问题。


# 后记

回到开篇，封装到底封装的是什么，我相信答案就在这篇文章出现频率最高的两个关键词里-那就是**变化**和**细节**，Bridge模式只是在特定场景下应用的一种封装技巧。
