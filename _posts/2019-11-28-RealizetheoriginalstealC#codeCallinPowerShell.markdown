---
layout: post
title: "实现原样偷C#代码，在PowerShell中调用"
tags: PowerShell进阶学习
---
![image](http://ny9s.com/picupdate/20191128111744.png)
很早以前用过直接调用别人写好的DLL，直接在PowerShell中使用，知其然不知其所以然，为了防止以后忘记，这次做个详细记录。
# 引子
PowerShell不是万能的，但毕竟都是.Net家族的儿子，很多系统原生无法实现的功能（特别是图形编程，以及传统领域的操作），大部分都能在'C#'那找到合适的例子，所以或多或少算是有个解决方法。

现在最大的挑战是，PowerShell毕竟出来这么多年，马上都要[7.0](https://devblogs.microsoft.com/powershell/)了，因为方法的变更，C#代码在PowerShell中直接照抄，有时候会报错。为此还要去爬techet翻官方文档。

所以即便是能抄C#代码，也不代表这是个简单的活，它至少需要两项能力
1. 简单看懂C#，知道C#的操作方法和语法，还有一些作者的习惯性命令简写。
2. PowerShell要溜，因为要做一个C#到PowerShell的翻译。

老实说，第二项是最操蛋的。毕竟是个翻译工作，之前也在[如何更智能的看懂PowerShell的英文注释](http://github.ny9s.com/2019/10/TransitPowershellCodeUseMicrosoft/) 中表述过类似的概念。
 
# DLL的优势
 DLL的优势是方法已经由作者写死了，用起来和模块一样，但是理论上，速度更快一些。这适合有些作者没有开源代码的情况来使用。而且成型的DLL，用起来真的是非常简单。
 
 像最常用的DLL，一个是汉字转拼音的微软官方DLL [Microsoft Visual Studio International Pack 1.0 SR1](https://www.microsoft.com/zh-cn/download/confirmation.aspx?id=15251)，一个就是下面例子的QQ数据库了。
 ![image](http://ny9s.com/picupdate/20191128111744.png)
 
# 实现
我的想法很简单，就是找到C#的例子后，如果代码相对很复杂，做C#到原生PowerShell的转换很麻烦的话，可以先用`Visual Studio`生成DLL，然后直接使用。

首先还是学习了一下老司机的文章
[在.NET中的C# DLL文件的生成与使用](https://blog.csdn.net/tmylzq187/article/details/51783633/)

照着大哥的方法实现了一遍，首先生成个项目，然后写入代码

![image](http://ny9s.com/picupdate/20191128113614.png)
 
 接下来正常的生成DLL，然后用下面的方法调用DLL
 
 ![image](http://ny9s.com/picupdate/20191128113815.png)
 
 关于这个模块中有哪些可以用的方法，在下拉菜单中可以看到。
 
 ![image](http://ny9s.com/picupdate/20191128113847.png)

这是一个相对较复杂的代码，里面方法就很多了。

![image](http://ny9s.com/picupdate/20191128114016.png)
 
 这个时候查询，也能找到相应的方法。
 
 ![image](http://ny9s.com/picupdate/20191128114122.png)
 
# 总结
 通过C#转DLL，然后PowerShell调用DLL的方法，极大的降低了从C#转换到PowerShell的难度。基本不需要对C#的阅读能力了。
 ![image](http://ny9s.com/picupdate/20191128114356.png)
 
