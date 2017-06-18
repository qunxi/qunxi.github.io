---
author: qunxi
create: 2017-06-07 18:43+08:00
update: 2017-06-07 18:43+08:00
layout: page
title: "设计模式-Composite模式"
description: ""
comments : true
categories:
tags:
- 设计
- 模式
---

# 前言 

组合是面向对象最常用的代码复用形式，我们可以将多个对象组合成更加复杂的对象。今天我们将介绍另一种结构模式-Composite模式，虽然翻译过来也叫组合，但是这里提到的组合并没有前面的宽泛。下面我们来看看这种模式。
<!--more-->

# 1. 树形组合 

在现实世界中，事物组合关系错综复杂很难找到唯一的规律或模式来描述它们。但是我们可以根据不同组合的自身特点，将它们归纳成不同的种类，比如线性的序列、非线性的图、树等组合方式。 而Composite模式就是用来处理树形结构的。

下面我们来看下树形结构的定义：

 > 树形结构是一种层次的嵌套结构。 一个树形结构的外层和内层有相似的结构， 所以这种结构多可以递归的表示

在现实世界中树形结构随处可见，它已经成为归纳分类的重要工具，比如企业组织架构、数据组织方式(XML)、思维导图等，都是树形结构的抽象。

在使用计算机时我们习惯将内容或功能相关的文件放在同一个的目录下，然后再将相关目录包含在一个更大主题的目录下，这样不仅方便查找，更重要的是它符合人类记忆习惯-人总是喜欢用归纳的方式辅助记忆。下面我们将简化后的文件系统设计如下：

```
public abstract class File
{
    public long FileId{get; private set;}
    public abstract void Open();
    public abstract void Close();
}

public class BinaryFile ：File{
    ...
}

public class TextFile : File{
    ...
}

public class Folder
{
    ...

    public void AddFile(File file){
        _files.Add(file);
    }
   
    public void RemoveFile(File file){
        _files.Remove(file);
    }
    
    public int GetFilesCount(){
        return _files.Count;
    }

    private List<File> _files;
}
```

在上面的例子中，**Folder**和**File**是独立的对象，**Folder**更像是管理**File**的容器。不要忘了**Folder**是可以自包含的，那么这个时候我们就要对**Folder**和**File**做区别处理。下面是是对**Folder**类的进一步扩展。

```
public class Folder{
    ...
    public void AddFolder(Folder folder){
        _childFolders.Add(folder);
    }

    public vid RemoveFolder(Folder folder){
        _childFolders.Remove(folder);
    }

    public int GetFilesCount()
    {
        int count = _files.Count;

        foreach(var folder in _childFolders){
            count += folder.GetFilesCount();
        }
        return count;
    }

    private List<Folder> _childFolders;
    ...
}
```
我们添加了额外的目录处理函数和成员变量，看上去好像很完美，但是我们必须看到是因为我们的例子比较简单，如果文件和目录还有其他的行为操作，那么**Folder**类将变得更加复杂。下面我们看看Composite模式如何解决之一问题的。


# 2. Composite模式的重构

在开始重构之前我们先看看Composite模式的定义：

> 将对象组合成树形结构以表示**部分-整体的层次结构**。 Composite使得用户对单个对象和组合对象的**使用具有一致性**。

根据定义，我们提取出前面例子中符合Composite模式的特点：

1. 整体部分的树形层次结构：`GetFilesCount`函数的递归调用体现了Folder和File的嵌套层次关系
2. 整体部分的行为一致性： Folder和File拥有相识的行为操作，比如`Open`、`Close`等。

下面我们将**File**提取成一个抽象基类。这样我们就不需要在**Folder**中区别处理目录和文件两种成员。

```
public abstract class File
{
    public long FileId{get; private set;}
    public abstract void Open();
    public abstract void Close();
}

public class TextFile : File{
    ...
}

public class BinaryFile : File{
    ...
}

class Folder : File{
    ...

    public void  AddFile(File file){
        _files.Add(file);   
    }

    public void RemoveFile(File file){
        _file.Remove(file);
    }

    public int GetFilesCount()
    {
        int count = 0;
        foreach(File file in _files){
            Folder folder = file as Folder;
            if(folder != null){
               count += file.GetFilesCount(); 
            }
            else{
                ++count;
            }
        }
        return count;
    }

    private List<File> _files;
}
```

# 3. Composite模式的透明性处理

上面的例子中，**Folder**类不再区分添加、删除的是文件还是目录，而且因为文件和目录都是继承自同一个抽象类**File**，所以**Folder**类也不再需要维护两份数据。但是在使用过程中我们还是发现很多地方我们需要分别处理文件和目录，比如客户在获得**File**引用时，我们需要单独的判断这个基类的具体事例是否是**Folder**对象，然后决定是否可以添加或删除文件。同样在`GetFilesCount`函数中我们以if条件区分了**Folder**和**File**的不同行为。

客户代码如下：

```
static void Main(){
    List<File> files = new List<File>();
    
    Folder folder = new Folder();
    File targetFile = new BinaryFile();
    folder.AddFile(targetFile);

    files.Add(folder);
    files.Add(new TextFile());

    // 删除所有文件夹下的指定文件 
    foreach(var file in files){
        var folder = file as Folder;
        if(folder != null){
            folder.RemoveFile(targetFile);
        }
    }
}
```

如何做到具体对象对客户来说更加透明？答案就在Composite的定义中，我们要做到不同层次对象的使用一致性，不论是目录还是文件。下面我们将`AddFile`、`RemoveFile`和`GetFilesCount`函数移到抽象类中，虽然这些函数对于文件来说好像有些不合理（文件如何添加文件...），但是有时默认的**空**行为，也未曾不是一个好的方法，因为一致的行为使得处理更加简单，而空方法也为带来任何的副作用。下面是Composite模式透明处理方式：

```
public abstract class File{
    public long FileId{get; private set;}
    public abstract void Open();
    public abstract void Close();
    public virtual void AddFile(File file){
        return;
    } 
    public virtual void RemoveFile(File file){
        return;
    }
    public virtual int GetFilesCount(){
        return 0;
    } 
}

public Folder : File{
    public void  AddFile(File file){
        _files.Add(file);   
    }

    public void RemoveFile(File file){
        _file.Remove(file);
    }

    public int GetFilesCount(){
        int count = 0;
        foreach(var file in _files){
            count += file.GetFilesCount();
        }
        return count;
    }
}

public static void Main(){
    
    List<File> files = new List<File>();
    
    Folder folder = new Folder();
    File targetFile = new BinaryFile();
    folder.AddFile(targetFile);

    files.Add(folder);
    files.Add(new TextFile());

    // 删除所有文件夹下的指定文件 
    foreach(var file in files){
        file.RemoveFile(targetFile);
    }
}
```

![composite pattern](/post-images/2017_6_18_composite_pattern.png)

现在代码使用更加简单了，没有了任何的`if`判断。客户在也不需要区分**Folder**还是**File**。当然任何的便利都不可能是免费的，它必须牺牲一些条件作为交换。比如抽象基类行为定义的膨胀；还有出于安全考虑，有些约束性的条件如果用户调用了，系统需要给出警告或提示，那么上面的空行为，显然是不满足需求的。所以设计其实就是一个平衡的过程，它没有对错，只有适不适合当前的场景。

# 后记

Composite模式在类图结构上和[Decorator模式](https://qunxi.github.io/2017/05/25/design-pattern-decorator.html)非常相似，但是它们的目的完全不同。Decorator模式是为了在不添加新继承类的情况下动态给对象添加新的功能和职责。而Compsoite模式更强调的是嵌套层次对象统一的处理方式。

