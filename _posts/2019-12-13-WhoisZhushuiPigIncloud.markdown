---
layout: post
title: "谁是注水猪，MiB与MB的区别"
tags: PowerShell进阶学习
---
![image](http://ny9s.com/picupdate/20191213103508.png)
 
# 多的一个i
> 现在很多云厂商的参考文献中，开始越来越多的对存储和流量的计算单位使用`XiB`,例如`KiB`，`GiB`。例如下面这种描述，这种描述有没有猫腻呢？
[Azure 订阅和服务限制、配额和约束条件](https://docs.microsoft.com/zh-cn/azure/azure-subscription-service-limits)

![image](http://ny9s.com/picupdate/20191213101634.png)
 

一个小小的`i`,之所以产生这种差异，主要是历史原因，硬盘厂商计算存储使用1000进位，而计算机中则是1024(2的10次方)进位，在KB/GB/TB和KiB/GiB/TiB的转换中，`每一级都要经受一次转换的损失。`
```
MiB和MB，KiB和KB等的区别:
1KB(kilobyte)=1000byte,    1KiB(kibibyte)=1024byte
1MB(megabyte)=1000000byte, 1MiB(mebibyte)=1048576byte

```
 所以12TB的硬盘只有`10.91TiB`，缩水的1T多就是这么来的。
 但是按照现有计费方式，`12T（TiB）`的硬盘，实际要硬盘厂商制造一个`13.19T（TB）`的硬盘的，这多出来的肯定没人愿意买单
# 计算范例

```powershell
Write-Host -BackgroundColor Green 实际1GB的字节数 $(1GB) -NoNewline
Write-Host -BackgroundColor red $(1GB)
Write-Host -BackgroundColor Green 按照1000进制计算，真实拥有1GiB，转换成GB的数量  -NoNewline
Write-Host -BackgroundColor red $(1GB/1000/1000/1000)
Write-Host -BackgroundColor Green 硬盘厂商眼中的1GB，实际的GiB大小   -NoNewline
Write-Host -BackgroundColor red $(1000*1000*1000/1GB)

```
![image](http://ny9s.com/picupdate/20191213103010.png)

# 云厂商的计算方式
值得~~吐槽~~`开心`的是，至少三家A开头的云厂商都是使用`注水猪没有i`的计费方式。所以客户是不需要担心什么的，因为这很公平。但是如果客户查看了账单有疑问，你可能需要给客户解释一下，为什么`多收了三五斗`


![image](http://ny9s.com/picupdate/20191213105523.png)
 

 ![image](http://ny9s.com/picupdate/20191213101038.png)
 

![image](http://ny9s.com/picupdate/20191213101003.png)
 
