---
layout: post
title: "PowerShell快速将多行字符串转成对象"
tags: PowerShell初阶学习
---
# 背景
写脚本的时候，总有临时想构造一堆数组的时候，正经的一个数组是下面这样的
```powershell
$a=1,2,3,4,5
$a="aa","bb","cc"
```
纯数字的数字比较简单，用逗号分隔即可，字符串的对象就要前后带引号，以及逗号分隔，相对而言，构造起来不那么`优雅`

# 转换
可以使用下面这种方式，快速转换这种关系
```powershell
$aa="张三
李四
王二
麻子"
($aa -split "\r?\n").Trim()
```
这样转换出来的结果就是一行一个数据，具体的应用场景可以是测试、或者比较短小的配置文件。
