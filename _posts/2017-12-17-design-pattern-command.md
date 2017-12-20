---
author: qunxi
create: 2017-12-17 18:43+08:00
update: 2017-12-17 18:43+08:00
layout: page
title: "设计模式-Command模式"
description: ""
comments : true
categories:
tags:
- 设计
- 模式
---
# 前言

在介绍[Strategy模式](https://qunxi.github.io/2017/10/17/design-pattern-strategy.html)时我们提到Command模式与其非常相似，尤其是UML图示。但是从意图出发，两者却有很大的区别，今天我们一起来了解下Command模式。
<!--more-->

# 1. 行为请求和实现的解耦

在开发过程中我们经常将命令与其实现绑定在一起，理由很简单因为相关性大。但是这很容易导致一些习惯性的耦合，比如实现一个绘图工具，我们需要通过不同的菜单项，画出不同的几何图形。下面是我们最直接的实现方式：

```C#
class Menu
{
    // ...
    private Graphic _graphic;
    private Pen _selectedPen = new Pen(Color.Black, 3);

    public void DrawTriangle(List<Point> points)
    {
        Debug.Assert(points.Count == 3);
        this._graphic.DrawPolygon(this._selectedPen, points.ToArray());
    }

    public void DrawCircle(Point center, double radius)
    {
        this._graphic.DrawEllipse(this._selectedPen, center.X - radius, center.Y - radius,
                      radius + radius, radius + radius);
    }

    public void DrawRectangle(Point leftTop, Point rightBottom)
    {
        Rectangle rect = new Rect(leftTop, new Size(Math.Abs(rightBottom.X - leftTop.X), Math.Abs(rightBottom.Y - leftTop.Y));
        this._graphic.DrawRectangle(this._selectedPlan, rect);
    }

    private void DrawMenuItem(){
        // UI 画菜单项
        DrawTriangleMenuItem();
        DrawCircleMenuItem();
        DrawRectangleMenuItem();
        .....
    }
}
```

上面的实现有一个明显的问题-功能逻辑和`Menu`存在紧耦合。不论是想重用`Menu`还是重用功能逻辑都非常的困难，现在我们必须对代码进行重构。首先很容易想到的就是将功能逻辑提取成独立于`Menu`的类，我们将它命名为`Document`，最后让Menu去调用`Document`的功能逻辑，这是一种很常规的做法，而且也很容易想到。但是`Menu`对`Document`这种复杂对象的直接依赖也未必是件好事。比如我们要还原一些在程序异常崩溃前的操作命令亦或一些其他需要可追溯的请求功能时，如果直接调用命令实现方法，我们很难获取这些方法的状态。那唯一的方法就是将这些操作命令定义成一个有状态的对象，这就是我们使用Command模式的目的。当然命令请求对象应该保持尽可能的小，它真正的实现逻辑并不包含在命令对象中，因为这可能导致大量的资源消耗，而且不利于功能逻辑的复用。

在开始使用Command模式进行重构前，让我们先来看看Command模式的标准定义:

> 将一个请求封装为一个对象，从而使你可用不同的请求对客户进行参数化；对请求排队或记录请求日志，以及支持可撤消的操作。

根据定义我们必须抽象出一个请求对象，下面就是具体的实现代码：

```C#
interface ICommand{
    void Execute();
}

class DrawRectangleCommand : ICommand{
    private Document _document;

    private Point _leftTop;
    private Point _rightBottom;
    private Color _foreColor;

    public DrawRectangleCommand(Color color, Point leftPoint, Point rightBottom){
        this._leftTop = leftTop;
        this._rightBottom = rightBottom;
        this._foreColor = color;
    }

    public void Execute(){
        _document.DrawRectangle(this._foreColor, this._leftTop, this._rightBottom);
    }
}

class DrawTriangleCommand : ICommand{
    private Document _document;

    private List<Point> _points;
    private Color _foreColor;

    public DrawTriangleCommand(Color color, List<Point> points){
        this._points = points;
        this._foreColor = color;
    }

    public void Execute(){
        _document.DrawPolygon(this._foreColor, this._points);
    }
}

class DrawCircleCommand : ICommand{
    private Document _document;

    private double _radius;
    private Point _center;
    private Color _foreColor;

    public DrawCircleCommand(Color color, Point center, double radius){
        this._center = center;
        this._radius = radius;
        this._foreColor = color;
    }

    public void Execute(){
        _document.DrawEllipse(this._forColor, this._center, this._radius);
    }
}
```

上面代码我们将所有的请求都抽象成具体的命令对象。而绘画几何图像形的实现由*Target*对象`Document`完成。当然命令触发的*Source*对象就是上面提到的`Menu`。通过`ICommand`的抽象给我们带来了以下好处：

* **空间的解耦**：所谓空间解耦有两点意义，首先我们将命令的请求和命令实现的逻辑进行了分离，方便功能逻辑的重用；其次因为命令请求变成了具体的对象，那么可以序列化命令使得分布式操作会变得水到渠成。

* **时间的解耦**：所谓时间解耦，其实就像回调函数，它不一定非要在调用函数的时候做出及时的相应，我们完全可以保存这些命令对象，等到我们真正需要时或某些条件满足时进行调用（比如延时操作）。

这两点在现实开发中非常有用，比如后面我们会谈到的可撤销的命令，以及在WPF中MVVM模式下的Command模式，都是利用这两点特性。

Command模式类图

![command_pattern](/post-images/2017_12_18_command_pattern.png)

# 2. 可撤销的操作

如果场景仅仅是上面画几何图形这种功能，那么使用Command模式显得有些繁琐，因为它不仅没有让我们切实体会到之前提到的好处，还增加了我们维护大量小对象的成本。下面让我们看一个相对应景的例子-命令的Undo和Redo。与之前例子不同的是，我们添加了一个Undo操作在`ICommand`接口中以及一个管理所有命令对象的`CommandMananger`类，代码如下：

```C#
interface ICommand
{
    void Execute();
    void Undo();
}

class DrawCircleCommand : ICommand{
    ...
    public void Undo(){
        this._document.DrawCircle(_document.BackgroudColor, this._center, this._radius);
    }
}

class DrawTriangleCommand : ICommand{
    ...
    public void Undo(){
        this._document.DrawPolygon(_document.BackgroudColor, this._points);
    }
}

class DrawRectangleCommand : ICommand{
    ...
    public void Undo(){
        this._document.DrawEllipse(_document.BackgroudColor, this._center, this._radius);
    }
}

class CommandManager{
    private Stack<ICommand> _undoCommands;
    private Stack<ICommand> _redoCommands;

    public void Undo(){
        if(this._undoCommands.Count == 0){
            return;
        }
        ICommand cmd = this._undoCommands.Pop();
        cmd.Undo();
        this._redoCommands.Push(cmd);
    }

    public void Redo(){
        if(this._redoCommands.Count == 0){
            return;
        }
        ICommand cmd = this._redoCommands.Pop();
        cmd.Execute();
        this._undoCommands.Push(cmd);
    }
}
```

因为Command模式将命令请求对象化，所以我们可以很方便的利用`CommandManager`来存储所有的请求，而且可以在需要时还原之前的所有请求。

# 3. Command模式在WPF的应用

Command模式在WPF（Windows Presentation Fundation）开发中是一个很重要的概念。WPF定义一个ICommand接口，只要实现了这个接口的自定义命令操作都可以直接绑定到控件的Command属性上（比如Button，MenuItem等）。比如在MVVM模式下，我们会在ViewModel中实现很多自定义的ICommand属性，但是因为ICommand的接口是固定的，所以我们完全可以实现一个通用的工具类-`DelegateCommand`。另外一些复杂操作可能包含多个操作组成，所以我们可以结合[Composite模式](https://qunxi.github.io/2017/06/18/design-pattern-composite.html)来实现一个`CompositeCommand`帮助我们完成命令的组合。以下是具体实现代码：

```C#
class DelegateCommand<T>: ICommand{

    private readonly Func<T, bool> _canExecute;
    private readonly Ation<T> _exectue;

    public DelegateCommand(Func<T, bool>canExecute, Action<T> execute){
        this._canExecute = canExecute;
        this._execute = execute;
    }

    public void Execute(T parameter){
        this._exectue(parameter);
    }

    public bool CanExecute(T parameter){
        return this._canExectue(parameter);
    }
    ...
}

class CompositeCommand: ICommand{
    ....
    private IList<Command> _commands = new List<ICommand>();

    public void RegisterCommand(ICommand cmd){
        this._commands.Add(cmd);
    }

    public void UnregisterCommand(ICommand cmd){
        this._commands.Remove(cmd);
    }

    public void Execute(){
        foreach(var cmd in this._commands){
            cmd.Execute();
        }
    }
    ...
}
```

# 后记

C#的[delegate](https://qunxi.github.io/2016/10/16/csharp-delegate.html)其实就是一个简易的Command模式。它本质是对相同函数签名的对象化，而命令请求的ICommand本质就是`Exectue`这个接口函数。这正好印证了一点模式是在弥补语言本身的缺陷，而随着语言的更新完善，很多模式也被慢慢的融入到语言的新特性中，但是思想殊途同归。