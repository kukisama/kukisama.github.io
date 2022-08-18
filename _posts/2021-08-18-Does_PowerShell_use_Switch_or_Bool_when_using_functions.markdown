---
layout: post
title: "PowerShell使用函数时采用Switch还是Bool？"
tags: PowerShell进阶学习
---

> 这是一篇翻译整理的稿子，相对而言比较有用，分享出来给大家。
>
> 原帖为 [Using PowerShell Switch vs. Boolean Parameters in SMA Runbooks](https://powershellmagazine.com/2013/12/20/using-powershell-switch-vs-boolean-parameters-in-sma-runbooks/)
>
> 原文发表在2013年，但是2021年依然还是有这个问题，或者说是这种设计。
>
> 看完原文只有一个感觉：解决问题生硬粗暴，完全就是解决不了问题，直接解决掉提问题的人的样子。但是还就是一点办法没有。

# 背景知识

我们在使用一些PowerShell命令的时候，会有一些`特殊的参数`，比如

```powershell
Out-File C:\aaa.csv -Append
```

这其中有没有`-appand`，对命令执行结果差异很大。（有的话表示文件追加，没有的话表示直接覆盖。）

这种参数还有一个特点就是，它`不需要`对其赋予一个特定的值，只要写入即可。

# 传统构造

传统构造的语句是这样的，可以看到能够实现和原生命令类似的效果。

```powershell
function MyFunction {
    param([switch]$A )
    if ($a -eq $true)
    { "真的" }
    else
    { "假的" }
}
#执行如下语句
MyFunction
假的
#执行如下语句
MyFunction -A
真的
```

# 问题现象

但是在`PowerShell workflow`中，且套用`两层workfolw`的前提下，会出现错误，导致无法继续。

![image](../assets/20210817152538.png)

# 解决方法

首先考虑是，`两层workflow`的情况常见么？从实际使用而言，真的很常见。如果想在函数或者`workflow`中使用这种参数传递，只能改用如下的方法，简言之就是将`switch`类型改为`bool`。但是用了这种方法后，`较为优雅的写法`也无法使用了。

在函数中的写法会有如下变动。

```powershell
function MyFunction {
    param([bool]$A )
    if ($a -eq $true)
    { "真的" }
    else
    { "假的" }
}
```

注意上方的写法中，`swtich`被修改为`bool`，而使用方法也变成了下面两个例子，在输入-A后，还需要加上$true的尾巴。

```powershell
#执行如下语句
MyFunction
假的
#执行如下语句
MyFunction -A $true
真的
```

而`workflow`的写法也会变成这个样子

```powershell
Workflow Invoke-NestedWorkflow
{
    Param([bool]$SomeSwitch)
    "I'm in the nested Runbook"
    $SomeSwitch
}

Workflow Invoke-ParentWorkflow
{
    "I'm in the parent Runbook"
    Invoke-NestedWorkFlow -SomeSwitch $true
}

Invoke-ParentWorkflow
```

# 问题影响

如果是用纯的`workflow`方法来写脚本，影响还是挺大的，因为较为直观的参数调用方式会被改变，像输出文件、记录日志这种经常用参数来控制的函数/workflow，`必须`做一定的修改才能执行。	

那么用`InlineScript {}`的方式能否跳过这个限制呢？答案是也是不行的，同样会失败。

所以解决方法唯一，使用`bool`来进行使用。

那么为什么会有这种问题出现呢？从表现来看，有可能是workflow的线程之间交互的一些限制，导致没有正确传参。

# 影响范围

观察了一下，影响范围应该是这样的，两者缺一不可。

- PowerShell的workflow脚本
- 需要从workflow传参

不受影响的部分：

- 所有传参在函数内部完成，不需要`workflow`层级操控。

因此脚本需要有意识的控制，要么修改写日志/输出文件的方法适配这种需求，要么控制不要在workflow去传递这种参数。

或者更简单，不用workflow自然碰不到这问题。

