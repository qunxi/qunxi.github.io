---
author: qunxi
create: 2016-11-27 18:43+08:00
update: 2016-11-27 18:43+08:00
layout: page
title: "设计模式-Decorator模式"
description: ""
comments : true
categories:
tags:
- 设计
- 模式
---
# 前言

在[上一篇文章](https://qunxi.github.io/2017/03/25/design-pattern-adapter.html)我们提到了Adapter模式，它可以很好的将已有的实现适配到不同的接口上，从而达到复用已有功能。但在很多项目中我们往往遇到的是因为设计之初的局限性，导致我们不得不在后期对已有功能进行扩展或修改。如何提供一种灵活动态的扩展和装饰能力？今天我们将介绍结构型模式中的另一种模式-**Decorateor模式**。

<!--more-->

# 1. 功能的扩展和修饰

在开始介绍Decorator模式前，我们依然引入一个具体的场景：假设我们有一套字符库，它包含了多种语言字符的实现。为了美化这些字符，我们希望为这些字符设计一些样式(比如宋体、楷体、宋楷等)。

以下是字符库的已有代码实现

```
public interface IFont{
    void Draw();
}

public abstract class Character : IFont
{
    public virtual void Draw(){
        // 样式呈现
    }

    public int Weight{ get; set; }

    public int Color{ get; set; } 
    ...
}

public class ChineseCharacter : Character{

}

public class EngilshCharacter : Character{

}
```
如何才能为已有字符呈现不同的样式呢？通常我们最先想到的就是下面两种方式：

1. 直接**修改**`EnglishCharacter或Chinese Character`。这种方法最简单，但也最容易破坏OCP(Open Close Principle)原则，所有字体的呈现方式要在每种字符类都重复的实现一遍。如果Character是来自一个没有源代码的第三方库，那么这种方法就行不通了。

2. **继承**`EnglishCharacter或Chinese Character`类，并重写`Draw`方法。这种方式解决了无法修改源代码的问题，但是它依然有自己的缺陷：继承是种静态的扩展和修饰，每个字符子类都是一种字体样式的呈现，它将样式和具体Character的核心功能绑定在了一起。如果字符需要呈现多种字体样式，我们就必须再添加另外一个子类，这不仅增加了类的纵向继承层次，让类变得庞大臃肿，而且还增加了类的数量。
下面是继承的实现方式：

```
public class SongKaiChineseCharacter : ChineseCharacter{

    public void Draw(){
     }
}

public class KaiChineseCharacter : ChineseCharacter{

    public void Draw(){
     }
}

public class SongChineseCharacter : ChineseCharacter{

    public void Draw(){
     }
}
```

当然上述两种方法并不是没有任何优势，如果字体样式和字符的呈现样式是相对固定的，那么使用继承可能更加简单，而且大大降低了管理成本。

除了上面两种方式，我们还有其他方式来扩展和装饰已有的实现吗？由于语言范式或特性的不同，我们就以C#为例，看看C#是如何扩展修饰已有类型的实现。

# 2. C#的扩展语法糖

和继承修改类似，C#分别提供了partial类和扩展方法来方便类型对象的扩展。下面我们看看这两种方法：

* partial：partial关键字其实是C#为对象修改提供的一个所谓的符合开闭原则(OCP)的语法糖，它可以在不破坏已有类的情况下，加入新的方法、成员甚至是继承关系，很多时候为了便于代码的管理，我们可以通过partial类来分离框架自动生成和用户自定义的代码。当然它在本质上和一般的修改没有太大的区别(编译器在编译代码时将相同的partial类合并一个)。当然它也有局限性，因为我们必须保证所有的partial类是在同一个程序集（Assambly）中，否则编译器不会将两个partial类合并。下面是partial的实现方式：

```
//  首先假设原来的Character类也定义成了partial类
public partial abstract class Character
{
    public void DrawSong(){
    }

    public void DrawKai(){
    }

    public void DrawSongKai(){
    }
}
```
* 扩展方法：扩展方法也是C#里头的一个非常常用的特性。它不再假设我扩展的方法必须和被扩展的类在同一程序集中，这样我们就可以扩展没有源代码的第三方类型。它不仅做到编写代码时的开闭原则，而且保证编译时的独立性。下面是扩展方法实现方式：

```
public class static class ChineseCharacterEx{
    public static void DrawSong(this ChineseCharacter cnChar){
        chChar.Draw();
        // 宋体样式
    }
    
    public static void DrawKai(this ChineseCharacter cnChar){
        chChar.Draw();
        // 楷题样式
    }
    
    public static void DrawSongKai(this ChineseCharacter cnChar){
        chChar.Draw();
        // 宋楷样式
    }
}

```
尽管如此，上面的方法依然有自己的不足，我们看到上面所有方法(继承除外)，我们很难对具体某个函数签名进行扩展和修饰。这就导致如果我们添加了某种字体的Draw方法时，我们就必须修改客端代码，而且同一种样式的呈现逻辑我们必须在不同字符重复实现多次。如何才能重用字体逻辑并修饰扩展已有的函数签名？下面我们开始介绍真正的主角-Decorator模式。

# 3. Decorator模式

在解决具体问题前，我们先看下Decorator模式的定义：

> 动态地给一个对象添加一些额外的职责。就增加功能来说， Decorator模式相比生成子类更为灵活。

定义中强调了Decorator模式的**动态**特性，以及和继承相比的**灵活性**。下面我们将字体的样式处理单独的提取成不同的类：

```
public abstract class Font : IFont{
      public virtual void Draw(){
      }
}

public class SongFont : Font
{
    private readonly IFont _character;

    public SongFont(IFont font){
        _character = font;
    } 

    public virtual void Draw(){
        font.Draw();
        // 添加自己的字体特性
    }
}

public class KaiFont : Font
{
    private readonly IFont _character;

    public KaiFont(IFont font){
        _character = font;
    } 

    public virtual void Draw(){
        font.Draw();
        // 添加自己的字体特性
    }
}
```
Decorator类与被修饰类继承了共同的接口，同时被修饰类作为构造参数传入Decorator类。以下是具体的类图关系。
![decorator pattern](\post-images\2017_5_25_decorator_pattern.png)

通过Decorator模式我们不仅内聚了Charater的核心功能，同时也将字体样式逻辑提取出来，变成一个可以复用的样式类。我们通过组合的方式构建出不同字符的不同样式，而不需要为每种字符实现所有字体。因为修饰功能和核心功能的相互独立，我们可以在不修改字符对象的前提下，灵活的添加和删除字体样式(OCP)。下面是Decorator模式和继承扩展的使用对比。

```
void Main()
{
    /**继承模式的应用**/
    // 宋体
    IFont songCnChar = new SongChineseCharacter();
    songCnChar.Draw();

    IFont songEnChar = new SongEnglishCharacter();
    songEnChar.Draw();

    // 楷体
    IFont kaiCnChar = new KaiChineseCharacter();
    kaiCharCn.Draw();

    IFont kaiEnChar = new KaiEnglishCharacter();
    kaiCharCn.Draw();

    // 宋楷
    IFont songKaiCnChar = new SongKaiChineseCharacter();
    songKaiCharCn.Draw();

    IFont songKaiEnChar = new SongKaiEnglishCharacter();
    songKaiCharEn.Draw();

    /**装饰者模式的应用**/
    // 宋体
    IFont songEn = new SongFont(new EnglishCharacter());
    songEn.Draw();

    IFont songCn = new SongFont(new ChineseCharacter());
    songCn.Draw();

    // 楷体
    IFont kaiEn = new KaiFont(new EnglishCharacter());
    kaiEn.Draw();

    IFont kaiEn = new KaiFont(new ChineseCharacter());
    kaiEn.Draw();

    // 宋楷
    IFont songKaiEn = new KaiFont(songEn);
    songKaiEn.Draw();

    IFont songKaiCn = new SongFont(kaiCn);
    songKaiCn.Draw();
}
```
Decorator模式和继承相比，提供了横向的扩展，这不仅大大减少了代码的重复，而且还增加了组合的灵活性。当然和所有模式一样，Decorator模式也是某一场景的权衡应用。它的缺点也很明显，因为逻辑分散到不同的类中，对于不熟悉整个系统的人来说变得不易理解和调试。而且生产了一定数量的小类，增加了管理的复杂度(这点是相对的，任何方法都有边际效益递增或递减的拐点)。另外在这个例子中没有被体现出来的缺点，如果你要修饰的不是IFont接口，而是庞大的Charater类，那么每一个修饰类的实现成本也是很大的，这个时候使用Strategy模式可能更加灵活，在之后的Strategy模式中我会再次提到这个问题。

# 后记

虽然装饰者模式提供动态扩展的能力，但是在设计初期我们不能让装饰者模式成为日后扩展就有余地的理由。我更倾向于轻设计重重构(**YAGNI原则(You Aaren't Gona Need It)**就是这个道理)。当然持续重构的意识需要团队文化的培养，以后的文章我会继续谈谈如何重构和团队重构意识的培养。

