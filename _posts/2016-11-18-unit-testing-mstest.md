---
author: qunxi
create: 2016-11-18 18:43+08:00
update: 2016-11-18 18:43+08:00
layout: page
title: "单元测试-实践篇(MsTest)"
description: ""
comments : true
categories:
tags:
- 工程
- 基础
---
# 前言

[上篇文章](https://qunxi.github.io/2016/11/05/unit-testing-basic.html)，简单的介绍了一下单元测试的基本概念，今天将结合具体的测试框架和例子来讲解怎么编写单元测试。这里我将使用MSTest框架（至于为什么用MSTest，没有原因，因为目前的项目在用，而且我认为所有的单元测试框架大同小异，我们讲授的是渔而非鱼）。
<!--more-->


# 1. 测试驱动开发(TDD)

如果你了解极限编程(XP)，应该非常熟悉测试驱动开发。他是有Kent Beck和Erich Gammak提出的一种实践方法

1. **运行失败(红):** 编写测试用例，因为没有具体的被测代码，所有测试用例都无法编译或者运行通过，测试框架通常以红色提醒。

2. **运行成功(绿):** 实现功能性代码，让编写的测试用例通过编译/运行，直到框架以绿色显示，表明所有测试用例都验证通过。

3. **重构:** 优化被测代码和测试用例。

测试驱动开发(TDD)是一种很不错的实践方法，对于之前没有尝试过的人来说，可能会有不适应('违背'了之前的开发流程)，但是它的确是一种高效的开发方式。当然测试只是所有开发流程中的一部分，至于用何种形式完成单元测试，这并不重要，我们的最终目的是保证代码的质量。下面我会以TDD的形式来演示如何编写单元测试。

# 2. 测试主体

在开始写单元测试之前我们必须有明确被测主体（就是我们的功能性代码），这样才能有的放矢。这里我们假设要实现一个BinarySearch的泛型类

```
public sealed class BinarySearch<T> where T : IComparable
{
    public int FindAny(List<T> list, T value)
    {
        return -1;
    }

    public List<int> FindAll(List<T> list, T value)
    {
        return null;
    }

    public int FindFirst(List<T> list, T value)
    {
        return -1;
    }
}

```
BinarySearch类的职责是对有序的数组进行时间复杂度为O(log2N)的查寻，如果找到就返回目标的索引，没找到就返回-1。这个类包含三个函数：FindAny(查询任意一个符合条件的索引位置)，FindAll(找得到所有符合目标的索引)，FindFirst(找的序列中符合条件的最小索引)。这里我们只是定义了函数的接口并没有具体的实现，下面我们将编写这个类的单元测试。因为篇幅的限制，我将以FindAny为例，其他函数的单元测试在设计与实现上应该是一样的。

# 3. 编写测试

## 0. 测试的基本规范

在面向对象的测试中，我们以被测类作为测试单元，然后针对具体的函数编写测试用例。下面代码是我们对FindAny编写的第一个测试用例。

```
[TestClass]
class BinarySearchTest
{
    [TestMethod]
    public void FindAny_TargetValueInTheList_ReturnCorrectIndex()
    {
        // arrange
        BinarySearch<int> search = new BinarySearch<int>();
        List<int> sortList = new List<int> { -1, 3, 5, 8, 9, 10 };
        int target = -1;
        int expectIndex = 0;

        // action
        int index = search.FindAny(sortList, target);

        // assert
        Assert.AreEqual(expectIndex, index);
    }
}

```

* 建立一个测试类(BinarySearchTest)：一般情况测试类以(被测类名 + Test后缀)命名，然后加上Attribute`[TestClass]`。通过测试类名我们可以了解这个测试类是针对具体哪个类进行测试。
* 根据不同场景编写测试用例: 测试方法往往以(被测函数名 + 测试场景 + 预期结果)命名，然后加上Attribute`[TestMethod]`。比如例子中的`FindAny_TargetValueInTheList_RreturnCorrectIndex`，我们一看到函数名就可以知道测试函数具体的功能与测试职责。
* 测试步骤(AAA): 编写测试用例时，我们一般分三步，首先(Arrange)根据测试场景组装测试数据和期望值，其次(Action)调用被测函数，最后(Assert)比较期望值。

作为开发人员我们应该像编写产品代码一样编写单元测试，好的单元测试就像一份文档，简洁明了，好的单元测试应该有一致的规范和标准，而且测试代码几乎没有复杂逻辑，任何复杂逻辑的出现说明你的被测主体关系复杂非常难用，有必要进行重构。

有时高覆盖率会使得测试类非常庞大，使得测试代码不易维护，这时我们可以使用C#的`partial` class在代码上进行拆分，同时使用`[Category("Group")]`Attribute在测试运行时进行分组(我们可以有选择，有针对性的运行测试用例)。

```
// BinarySearchTest_FindAll.cs 这个文件主要用于测试FindAll函数，但测试类名不变
[TestClass]
partial class BinarySearchTest
{
    [TestMethod]
    [Category("FindAll")]
    public void FindAll_TargetValueInTheList_ReturnCorrectIndex(){} 
}

// BinarySearchTest_FindAny.cs 这个文件主要用于测试FindAny函数
partial class BinarySearchTest
{
    [TestMethod]
    [Category("FindAny")]
    public void FindAny_TargetValueInTheList_ReturnCorrectIndex(){
    }

    [TestMethod]
    [Category("FindAny")]
    public void FindAny_TargetValueNotInTheList_ReturnMinusOne(){
    }

    [TestMethod]
    [Category("FindAny")]
    [ExpectedException(typeof(NullReferenceException))]
    public void FindAny_NullList_ReturnMinusOne(){
    }

    [TestMethod]
    [Category("FindAny")]
    public void FindAny_TargetValueNotInTheList_ReturnMinusOne(){
    }
}

```
私有(`private`)函数是否也要编写单元测试？我个人认为没有必要，但是在公有(`public`)函数的单元测试中必须要覆盖到私有函数的逻辑。另一个不是理由的理由，既然单元测试作为一份对外的文档，那么我们没有必要暴露被测主体的封装部分。

## 1. 测试用例编写

编写测试前，我们应该先设计测试用例，一般情况下我们需要cover一些简单使用场景，比如正确的函数状态、行为、异常错误处理、边界条件等。比如之前写的`void FindAny_TargetValueInTheList_ReturnCorrectInde()`就是测试函数正确行为的测试用例。当然我们编写的是模板类，我们还应该测试不同的数据类型，比如`float，double，char`，甚至是自定义类型。以下我们测试的是int类型，其他类型方法相同。我们可以用`[Category("Int Case")]`来给测试用例进行分组。

测试场景1：查询对象不在数列中

```
[TestMethod]
[Category("Int Case")]
public void FindAny_TargetValueNotInTheList_ReturnMinusOne()
{
     // arrange
     BinarySearch<int> search = new BinarySearch<int>();
     List<int> sortList = new List<int> { -1, 3, 5, 8, 9, 10 };
     int target = -1;
     int expectIndex = 0;

     // action
     int index = search.FindAny(sortList, target);

     // assert
     Assert.AreEqual(expectIndex, index);
}
```

测试场景2：在空数组中查找

```
[TestMethod]
[Category("Int Case")]
public void FindAny_EmptyList_ReturnMinusOne()
{
    // arrange
    BinarySearch<int> search = new BinarySearch<int>();
    List<int> sortList = new List<int>();
    int target = 4;

    // action
    int index = search.FindAny(sortList, target);
    int expectIndex = -1;

    // assert
    Assert.AreEqual(expectIndex, index);
}
```

测试场景3：数组为null的情况 

```
[TestMethod]
[Category("Int Case")]
[ExpectedException(typeof(NullReferenceException))]
public void FindAny_NullList_ReturnMinusOne()
{
    // arrage
    BinarySearch<int> search;
    int target = 4;

    // action
    int index = search.FindAny(search, target);
}
```

最后一个测试中`[ExpectedException]`Attribute是用来捕获异常预期的。这里我们在被测代码中没有捕获任何处理，但是单元测试可以告诉用户在使用被测代码时需要考虑到数组为`null`的情况，而且客户代码有义务捕获这个异常

## 2. 实现功能代码

编写完测试用例，我们需要实现被测函数，好让上面的测试用例编译运行通过

```
public int FindAny(List<T> list, T value)
{
    int heigh = list.Count;
    int low = 0;  

    while (heigh > low)
    {
        int mid = (heigh + low) / 2;

        if (list[mid] == value)
        {
            return mid;
        }
                
        if (list[mid] > value)
        {
            heigh = mid - 1;
        }
        else
        {
            low = mid + 1;
        }
    }

   return -1;
}
```

## 3. 测试代码的重构

* **代码覆盖**

为了测试的完整性，在我们完成实现后，我们需要根据具体的实现添加一些在一开始没有考虑到的测试用例。比如是否所有的路径都被覆盖到，之前的几个测试用例其实并没有完全覆盖所有的路径，甚至简单的行覆盖都没有做到。也许你已经发现在`FindAny_TargetValueNotInTheList_ReturnMinusOne()`和`FindAny_TargetValueInTheList_ReturnCorrectIndex()`中虽然我们得到了正确的结果，但是他永远不会进入`while`循环中的`else`判断条件，所以我们并不能知道是否可以正确的查找到数组靠右侧的数字。

```
[TestMethod]
[Category("Int Case")]
public void FindAny_TargetValueInRightOfList_ReturnMinusOne()
{
     // arrange
     BinarySearch<int> search = new BinarySearch<int>();
     List<int> sortList = new List<int> { -1, 3, 5, 8, 9, 10 };
     int target = 8;
     int expectIndex = 3;

     // action
     int index = search.FindAny(sortList, target);

     // assert
     Assert.AreEqual(expectIndex, index);
}
```
其实代码覆盖有很多，比如[行覆盖，条件覆盖，判定覆盖，多路覆盖](https://en.wikipedia.org/wiki/Code_coverage)等。对上述覆盖的了解有助于你写出覆盖率更高的代码。当然要做到100%的覆盖率还是很有挑战的。

* **添加修改单元测试**

根据上述的测试，在大多数使用场景好像都没什么问题。可以比如`0.15462839`和`0.15462833`作为`float`类型时是相等的，但是作为`double`类型时却是不等的。因为`float`的精度只有7位，第八位的比较已经不再有效，所以`float`和`double`一般不能简单的用==进行比较。既然之前没有考虑到这种场景，那么我们需要对这个场景编写单元测试来覆盖。一般情况我们可以添加一个Epsilon作为容忍度，但是基本类型(Double，Float)我们如何重载CompareTo函数？作为通用算法，我们是不能随便为上层业务强制定义精度的，这个应该由客户自行定义，不然通用算法就会变得不灵活。下面我们重新重构了被测代码。

```
 public int FindAny(List<T> list, T value, Func<T, T, int> Compare)
 {
    int heigh = list.Count;
    int low = 0;
    {
       int mid = (heigh + low) / 2;

      if (Compare(list[mid], value) == 0)
      {
            return mid;
      }

      if (Compare(list[mid], value) > 0)
      {
            heigh = mid - 1;
      }
      else
      {
            low = mid + 1;
      }
 }
```

```
[TestMethod]
[Category("Float Case")]
public void FindAny_DoublePrecise_ReturnMinusOne()
{
     // arrange
     BinarySearch<float> search = new BinarySearch<float>();
     List<float> sortList = new List<float> { -1f, 3f, 0.15462839f, 8f, 9f, 10f };
     int target = 0.15462836f;
     int expectIndex = -1;

     // action
     int index = search.FindAny(sortList, target, this.CompareFloat);

     // assert
     Assert.AreEqual(expectIndex, index);
}

private int CompareFloat(float a, float b)
{
    const float Epsilon = 0.0001f; 
    
    if (Math.Abs(a - b) > Epsilon)
    {
        return 0;
    }

    if (a > b)
    {
        return 1;
    }

    return -1;
}
```
上面我们将比较过程定义为一个委托传入被测函数，同时我们也添加了新的单元测试加以覆盖。还没结束，最后一步我们需要做的就是对之前的测试用例进行更新，因为之前的测试用例已经不能运行成功。这就是我们重构的基本流程：在修复一个defect之后添加新的测试用例来cover这个场景，如有必要还要更新之前编写的单元测试。

* **优化测试代码**

有时候我们需要构建一些繁琐的测试数据，而且这些数据又会被每个测试用例使用。如果我们重复在每个测试用例的Arrange部分进行初始化，很显然会有很多重复的代码，一旦以后变换了需求，那么我们需要多处修改初始化代码。很多测试框架都提供了类似MSTest中提供的[TestInitialize]，[ClassInitialize]这样的Attribute来完成资源构建的复用，同时如果有些资源需要在测试用例验证完毕后销毁，我们也可以用`[TestCleanup]，[ClassCleanup]`来标记。`[TestInitialize]，[ClassInitialize]`的区别：`[TestInitialize]`是在每一个测试用例启动时被调用，而`[ClassInitialize]`是在整个测试类启动时被调用。大多数情况我们应该使用[TestInitialize]因为每一个测试用例的数据应该尽可能的独立，以免多个测试用例运行后数据被污染。好的测试用例也应该是相互独立，而不应该有顺序的依赖。`[TestCleanup], [ClassInitialize]`同理，我们应该在每个测试用例结束后清理数据，以免影响下个测试用例。

```
BinarySearch<int> _search;

[TestInitialize]
public void TestSetup()
{
    _search = new BinarySearch<int>();
    // 初始化每个用例的功用资源或对象 
}

[TestCleanup]
public void TestCleanup()
{
    // 在测试用例调用后删除数据
}
```

# 后记

代码覆盖率一直是争议的焦点，多少的覆盖率是合理的，80%，90%还是100%。其实我觉得这个问题既好回答，又难回答。在排除为了考核而追求覆盖率的情况下，那么测试覆盖率当然越高越好。但在参差不齐的工作环境中，40%不一定比80%差(有时数据就是内行忽悠外行的高级工具，而且屡试不爽)。一刀切的设置一个比合理高一点的目标未必不是一件好事(万一实现了呢)，目标定了，然后具体问题具体分析(在追求卓越的道路上边际效益是递减的，所以平衡很重要)。

## （转载本站文章请注明[作者和出处](https://qunxi.github.io/)，请勿用于任何商业用途）