---
author: qunxi
create: 2016-12-27 18:43+08:00
update: 2016-12-31 18:43+08:00
layout: page
title: "软件全球化-Globalization"
description: ""
comments : true
categories:
tags:
- 工程
- 基础
---
# 前言

[上篇文章](https://qunxi.github.io/2016/12/18/g11n-unicode.html)介绍了软件全球化的基础-Unicode，Unicode的重大意义和秦始皇统一文字的意义一样伟大，在这个基础上我们再来谈谈作为程序员应该如何做好软件全球化。

<!--more-->
# 1. 软件的全球化

在写上一篇文章时，我就在纠结是否应该叫软件国际化，但是最终我还是决定叫软件全球化，因为在英文中软件的国际化和全球化是两个完全不一样的概念。下面我们来看看这几个概念

* 国际化(Internationalization = i18n)：一种应用软件的设计过程，这样设计的应用软件无需做大的工程改变就能适应不同语言和地区的需要。

* 本地化(Localization = l10n)：针对特定的市场，将软件产品按照最终用户的使用习惯，需要以及需求进行转化和定制的过程。

* 全球化(Globalization = g11n)：处理软件本地化和国际化的过程。

> Note: 1.因为国际化、本地化、全球化的英文字母太长，所以我们习惯取英文字母的首尾字母+中间字符的个数表示。2.微软同样定义了上面三个概念，但它将国际化和全球化的概念进行了互换。

从上面的概念我们可以看出全球化的过程就是国际化 + 本地化。而本地化就是我们通常理解的为不同市场定制化需求（包括语言和用户使用习惯）。如何高效的实现软件的本地化，那么我们必须做到有很好的国际化(World-Ready)设计。如果所有软件本地化的版本都各由一份独立的代码来维护，那么可以想象开发，测试，翻译会多么混乱。所以作为开发人员，有一个软件国际化的理念和设计思路是非常有必要的。

# 2. 如何做好全球化

## 1. 使用Unicode开发

使用Unicode开发是非常有必要的，它是软件全球化的基础。可以想象如果每一个本地化版本都使用各自的编码标准，那么我们需要耗费多少人力物力。即使每个市场的回报远远高于你的投入，但在如今全球化的趋势下，软件跨区域跨语言的交互已经成为必然，而没有使用Unicode，会使得开发人员疲于奔命在不同语言编码的转换而不是针对特定需求的功能定制。软件每进入一个市场开发人员就会陷入本地化的焦油坑。了解Unicode请阅读[软件全球化-Unicode](https://qunxi.github.io/2016/12/18/g11n-unicode.html);

## 2. 团队培养

在开发软件一开始我们就应该灌输团队关于软件全球化的意识，否则直到真正发现需要对具体市场进行本地化时，那么我们将会花更大的成本去修改之前的设计和实现。还有重要的一点是，团队千万不要按本地化职能拆分，否则产品将会无意识的糅合进不同文化的特性，而且各个团队会主观的放大自己本地化功能。所以必须对整个软件全球化要有统一的设计和规范，同时培养团队成员文化中立的意识。

## 3. 代码做到文化中立

本地化的定制相对软件功能是变化的，不同的市场有特殊的语言和文化要求，在软件设计中我们有两个重要的观点叫**分装变化点**、**关注点分离**，所以我们应该将本地化的资源和代码分离，并将每种本地化功能或资源统一封装在一个独立模块。这样在我们本地化软件时，就不会影响主体功能和其他本地化模块。
同样我们不需要重复的测试通用的业务逻辑，而是专注于本地化的集成测试。

## 4. 全球化测试

即使目前的项目只针对一个市场，我们也要考虑到不同本地化的实现和测试。为了测试的高效，我们要确保测试的产品是基于一份代码，即使是不同本地化版本。

* 保证代码文化中立(World-Ready)，比如利用“Pseudo-Localized”来构造假的语言或资源，测试软件是否可以适应任何本地化的替换。

* 针对具体市场的本地化测试，确保翻译，显示以及使用符合当地文化和习惯。

* 对多语言文本处理的测试，来保证多语言数据处理不会导致数据丢失以及编码间转换的正确性。

# 3. .Net 软件全球化实践

Windows平台为软件全球化提供能很多简洁便利的解决方案，尤其是.Net平台，下面我们来看看.Net平台下几个重要的概念：

## 1. Culture

文化是除语言以外另一重要的本地化要素，比如日期，货币，字体，数字等，不同国家或地区都有自己的使用习惯。.Net定义了三种层级关系的文化类型：

* Invariant Cultures（不变的文化）：InvariantCulture是对文化完全不敏感的，它以英文为语言基础，但是不涉及任何国家或地区相关的文化。我们通过给CultureInfo传入String.Empty来构造InvariantCulture。

* Neutral Cultures（中立的文化）: NeutralCulture只和语言相关，但是不涉及国家地区相关的文化。比如中文会涉及到繁体和简体还有不同地区的香港，台湾，新加坡等地域文化，但是我们可以建立与语言相关但和地区文化不相关的中立文化。比如我们可以通过给CultureInfo传入`zh`来构造中文的NeutralCulture。

* Specific Cultures（特定的文化）: SpecificCulture是针对特定语言地域相关的文化。我们可以分别通过传递`zh-CN`和`zh-TW`来构造大陆和台湾的特定文化。

其中InvariantCulture是NeutralCulture的父文化，NeutralCulture是SpecificCulture的父文化。通过这个层级关系，.Net可以在没找到特定文化时自动匹配兼容的父文化。

文化相关的类型都定义在System.Globalization命名空间下，比如日期，数字，文本等元素。其中`CultureInfo`是个非常重要的类，.Net下的每个线程分别附带两个CultureInfo相关的属性，一个是`CurrentUICulture`另一个是`CurrentCulture`。前者是在运行时获取特定文化的资源数据，后者用于决定日期、数字、货币的转换格式以及和文化相关的文本比较，排序等行为。具体`CultureInfo`介绍请参看[MSDN](https://msdn.microsoft.com/en-us/library/system.globalization.cultureinfo.aspx)

## 2. Resources文件

很多时候程序员会自作聪明的为了省事，将和用户接口(UI)相关的字符用常量来保存在代码中或者在程序运行时自动生产，看似聪明的做法却给之后的本地化埋下隐患，作为一个翻译人员要找到分散在代码中需要翻译的字符，甚至一些运行时才能生产的字符更是困难。.Net建议我们将资源相关的数据（图片，文字等）保存在.resx文件中，每个资源都有一个唯一的Name，而具体需要本地化的信息则保存在Value里。为了方便翻译人员我们还可以添加Comments，来帮组翻译人员理解文本的上下文。.Net可以通过的`ResourceManager`类来加载相应的资源。下面是具体加载的代码

```
ResourceManager resMgr = new ResourceManager("ProjectName.Properties.Resources", typeof(Resources).Assembly);

//resx 字符资源的KeyName = "AboutInformation";
string aboutInfo = ResourceManager.GetString("AboutInformation", CultureInfo.CurrentUICulture);         

//resx 图片资源的KeyName = "CompanyLogo";
Bitmap logo = (Bitmap) Image.FromStream(rm.GetStream("CompanyLogo"));
```
我们可以将不同文化的资源文件通resgen.exe生成二进制文件，然后通过设置当前线程的CurrentUICulture属性，让线程在运行时寻找匹配的资源文件。

## 3. Encoding

Unicode编码的重要性前面已经提到过，在创建默认的cs文件，VisualStudio就会默认的添加`using System.Text`，可见这个命名空间的重要性，而编码就在这个命名空间下。我们可以通过Encoding类来处理不同编码之间的转换以及Code Pages和Unicode的编码相关操作。.Net默认是用UTF16编码的，如果我们要将数据通过网络传输，我们可以通过下面代码转换成UTF8。

```
public static string convertUTF16toUTF8(unicodeString)
{
    Encoding utf8 = Encoding.UTF8;
    Encoding unicode = Encoding.Unicode;

    byte[] unicodeBytes = unicode.GetBytes(unicodeString);
    byte[] utf8Bytes = Encoding.Convert(unicode, utf8, unicodeBytes);

    return utf8.GetString(utf8Bytes);
}
```
更多编码相关内容请参考[MSDN](https://msdn.microsoft.com/en-us/library/system.text.aspx)

## 4. Check list

1. 使用Unicode！Unicode！Unicode！重要的事情说三遍。不要让编码转换成了你的日常工作。

2. 做到代码和资源文件的分离，不要用const，hardcode资源相关的信息，将UI相关的信息统一放到一个资源文件中。

3. 为需要本地化的字符串添加注释，以便翻译人员理解字符串的使用场景，做到精确翻译。

4. 不要在不同场景重用字符串，有的语言同一个词语可以表示不同的意思在不同场景，但一些语言只能表示一个意思。

5. 千万不要在程序运行时，或用链接的方式构建需要本地化的字符串，这样不利于翻译人员的工作。

6. 使用系统提供的比较，排序方法来处理字符串，不同文化的字符串比较，排序规则是不一样的，比如中文可以按笔画排序。

7. 尽可能的给字符串预留足够的空间(多50%)，因为表达一样的意思，不同语言需要的文字长度不一样。

8. 不要使用特殊文化语意的字符串（比如俚语之类），这会给翻译人员带来不便。

9. 不要使用带有特殊种文化或宗教意义的图片，同样图片不要带有特定语言的文字。

10. 不要假设机器安装了所有字体或某种字体适合所有的语言文字。

11. 清晰定义年月日，不要用字符直接解析，不同国家和地区年月日显示的顺序是相反的。(09/08/2016可能表示九月八号也可能是八月九号)

12. 在UI布局或自定义控件的时候，用相对位置，不要使用绝对位置，同时考虑镜像问题。因为不同国家的文化不同，UI的布局也不一样。比如阿拉伯人习惯从右往左阅读。

13. 保存数据，应该以文化中立的格式保存，以免不同文化的版本处理数据出现歧义。

# 后记

作为程序员我们应该培养自己在工程上的思想和意识。就像软件全球化的考虑，从长远出发是利大于弊的。

## （转载本站文章请注明[作者和出处](https://qunxi.github.io/)，请勿用于任何商业用途）
