---
layout: post
title: "PowerShell处理IPMI的UEFI启动"
tags: PowerShell进阶学习
---
# 引子
[ipmitool.exe](https://github.com/ipmitool/ipmitool)是一个非常棒的开源工具，没有之一。
通过这个工具，我们可以很方便的批量修改服务器的属性，比如启动顺序，检查服务器上的磁盘、内存、硬件信息。

我这里之前用到的一个功能是，用ipmi去修改服务器的启动顺序，调整为“下一次从PXE启动”。

网上找的命令，XX机器很好用。但是XXXX就不好用，比如`国产良心联想`。问题原因经过分析，也很清晰了。主要是设置成PXE启动没问题，但是它是传统模式的PXE，而非UEFI的PXE。如果要是用UEFI模式的PXE，请看下面的例子。

```powershell
#支持UEFI引导的IPMI配置,使用IPMItools 1818
$global:ipmiuser = "USERID" 
$global:ipmipass = "PASSW0RD"
function ipmix($ip, $set) {
    $setx = "-I lanplus -H " + $ip + " -U $global:ipmiuser -P  $global:ipmipass " + $set  
    cmd /c "ipmitool.exe  $setx"
}
```
我测试了好些例子，下面都是比较常用的例子。

可以看到例子，实际上ipmitool并没有原生命令去处理这个问题，而是用RAW数据，硬往IPMI写数据实现的。

所以我有个疑问，第一个发明这个解决问题方法的大哥，是怎么找到的呢？
```powershell
#检查主机是否开机
ipmix $csv.ip[1] "chassis status"
#设置目标主机的下一次启动为 UEFI模式的PXE
ipmix $csv.ip[1] "raw 0x00 0x08 0x05 0xe0 0x04 0x00 0x00 0x00"
#清空目标主机下一次启动
ipmix $csv.ip[1] "chassis bootdev none"
#传统的设置下一次启动为PXE，如浪潮、H3C
ipmix $csv.ip[1] "chassis bootdev pxe"
#检查当前连接通道信息
ipmix $csv.ip[1] "channel info"
#检查硬件传感器
ipmix $csv.ip[1] "fru"
#检查内存传感器，可以确认内存插槽、温度
ipmix $csv.ip[1] "sdr type Memory"
#检查磁盘类设备
ipmix $csv.ip[1] "sdr type 'Drive Slot / Bay'"
#检查PCIE设备的插槽
ipmix $csv.ip[1] "sdr type 'Slot / Connector'"
ipmix $csv.ip[1] "sel elist"
ipmix $csv.ip[1] "sensor list" 
#检查电源使用情况
ipmix $csv.ip[1] "dcmi power reading"
#检查DCMI可用选项
ipmix $csv.ip[1] "dcmi discover"


```
