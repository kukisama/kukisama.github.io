---
layout: post
title: "戴尔服务器的IPMI与裸金属部署"
tags: PowerShell进阶学习
englishname: Dell_IPMI_Bare_metal_deployment
---
![image](http://ny9s.com/picupdate/20200827215217.png)
 
# 需求
最近手里有一批戴尔的R740XD服务器需要批量装系统。以前针对华为浪潮中兴联想的机器，都可以通过开源的IPMI工具`ipmitool`来进行`批量`配置。实际上 也只是借用了两个功能
1. 配置系统使用UEFI启动模式
2. 配置系统的`一次性引导`（仅下一次重启时选择的启动项目，例如光驱、U盘、PXE或者本地磁盘、BIOS界面等）为PXE启动，默认从硬盘启动。

问题在于戴尔的做法有少许不同，导致实现起来花了很多心思。这就有点象吃惯了萝卜白菜（没有什么定制化）的IPMI管理界面，突然来一顿四川火锅，上下实在都受不了。

这里讨论的都是`批量`的前提，如果单纯是手工点选，任何厂商都没有任何差异
# 什么是IPMI
智能平台管理接口（Intelligent Platform Management Interface）原本是一种Intel架构的企业系统的周边设备所采用的一种工业标准。IPMI亦是一个开放的免费标准，用户无需支付额外的费用即可使用此标准。 

IPMI 能够横跨不同的操作系统、固件和硬件平台，可以智能的监视、控制和自动回报大量服务器的运作状况，以降低服务器系统成本。

### 简言之，IPMI是一个微型的操作系统，用来管理真实的物理设备

像远程KVM，开关机、配置BIOS这些都可以做。特别是大部分的IPMI都可以用命令行操作。

# 戴尔的IPMI

[官方帮助可以看这里](https://www.dell.com/support/manuals/cn/zh/cndhs1/integrated-dell-remote-access-cntrllr-8-with-lifecycle-controller-v2.00.00.00/racadm_idrac_pub-v1/racadm-syntax-usage?guid=guid-6cf590a1-f185-499b-b7c1-2a6d2fb958e6&lang=en-us)

 ![image](http://ny9s.com/picupdate/20200827213759.png)
 
 
 
 
戴尔这里其实真的是好心，重新设计了IPMI，包装成了iDRAC 这么一个东西，现在最新的版本已经是iDRAC9了。跟它适配的命令行IPMI工具就叫做`racadm`。当然这种集成的事情，华为和联想也都有完整产品。

获取这个工具的唯一前提或者说要求是，你需要一台戴尔服务器的`序列号`。然后去官网搜索下载
`适用于 Windows 的 Dell EMC Systems Management Tools and Documentation DVD ISO`，现在最新的版本现在是`v.9.4.0`

安装好后，可以提取出大概这么一个目录，我将IPMI工具也混合在了一起，接下来就可以干事了

![image](http://ny9s.com/picupdate/20200827213927.png)
 
# 痛点
 
~~问题不叫问题，叫~~==痛点==顾名思义，就是很难搞的地方。

先整理一下我们通过图形界面或者进入BIOS来实际要做的工作

- 重启计算机
- 进入BIOS
- 调整系统为UEFI模式
- 保存后再次重启，配置一次性启动为PXE，以及调整硬盘为第一启动设备。
- 之后的操作就是正常的通过PXE安装系统，过程不表。

通过racadm`get bios.BiosBootSettings.UefiBootSeq`可以检查现在UEFI的启动设备有哪些。

`get bios.BiosBootSettings.SetBootOrderEn`则可以列出哪些条目支持启动，比如网卡、硬盘。在这里它是以一个字符串，中间用逗号隔开，例如`RAID.Slot.6-1,NIC.PxeDevice.1-1`。最初的想法是用set的相关命令，将`RAID.Slot.6-1,NIC.PxeDevice.1-1`替换成`NIC.PxeDevice.1-1，RAID.Slot.6-1`，可惜的是这种方法==并不好用==。不仅如此，IPMI常用的`一次性引导`的配置也是有问题的。

另外还有一个问题，从BIOS切换到UEFI后，部分RAID卡在不开启`HddPlaceholder`的时候，是识别不到本地磁盘的。这个新的配置让打工仔的生活雪上加霜。

除此之外，IPMI的配置过程其实并不是实时生效，它相当于准备一个配置文件，然后再下次重启的时候，进入生命周期管理器中，把配置写入到BIOS中。从重启到修改结束，这个过程大概需要3到5分钟。

# 流程梳理
基于这个前提，重新设计IPMI的配置过程（纯命令实现）。
- 修改UEFI并设置计划任务，重启生效
- 修改`HddPlaceholder`，确认可以认到磁盘。默认情况下第一启动设备是PXE，正好符合我们的要求，到这里就可以开始装机了。
- PXE服务器开始裸金属推送。因为调整磁盘顺序的命令并不好用，所以采用一个折中的方法：启动设备只留硬盘，将PXE放到不允许启动的配置选项卡下。同时设置计划任务，不做任何修改，等下次重启再进行修改。
- 这里最最关键的地方在于，配置磁盘启动顺序不能是立即配置，必须要等待PXE将服务器启动起来之后，然后进行配置。

# 代码实现

加一点点细节就可以了。

为了提升代码可读性，将部分代码存在模块里面。同时因为戴尔做了二次封装，导致命令执行起来很慢，这里用到了之前我的文章介绍过的[多线程技术](http://github.ny9s.com/2020/01/PowerShellCheckDiskFreeSpace/)

首先是模块
```powershell
function PSMultithreading ($Throttle, $allnodename, $ScriptBlock) {
    #线程，需要操作的对象，脚本块
    $RunspacePool = [RunspaceFactory]::CreateRunspacePool(1, $Throttle)
    $RunspacePool.Open()
    $Jobs = @()
    $allnodename | ForEach-Object {
        $Job = [powershell]::Create().AddScript($ScriptBlock).AddArgument($_)
        $Job.RunspacePool = $RunspacePool
        $Jobs += New-Object PSObject -Property @{
            Server = $_
            Pipe   = $Job
            Result = $Job.BeginInvoke()
        }
    }
    Write-Host "请等待.." -NoNewline
    Do {
        Write-Host "." -NoNewline
        Start-Sleep -Seconds 1
    } While ( $Jobs.Result.IsCompleted -contains $false)
    Write-Host "作业已完成" -BackgroundColor Green

    $global:Results = @()
    ForEach ($Job in $Jobs) {
        $global:Results += $Job.Pipe.EndInvoke($Job.Result)
    }  
}

function servers ($path, $matchx) {
    $csv = Import-Csv -Path $path 
    $Global:servers = $csv | Where-Object { $_.servername -MATCH $matchx } | Sort-Object servername 
    $servers | ForEach-Object { $_.servername + "," + $_.ip + "," + ((ping $_.ip -n 1 -w 50) -match "Reply from .*") }
}

function DELLipmi  ($ip, $Scripts) {
    $IPMItemp = 'C:\racadm\racadm.exe  -u root -p calvin -r ' + $ip + " " + $Scripts
    cmd /c $IPMItemp
}

function WH1 ($servername, $ip ) {
    #Write-Host $servername -NoNewline -ForegroundColor Red
    #Write-Host $ip -NoNewline -BackgroundColor DarkGreen
    Write-Host  (($IPMIoutput -match "=")[-1]) -NoNewline -ForegroundColor Green
    Write-Host ""   
    $($servername + ',' + $ip + ",")
}

function IPMICheckAndSet ($code) {

    $IPMIoutput = DELLipmi $ip  $code 
    ($IPMIoutput -match "=")[-1] 
 
}

function DELLOutputInfomation ($count, $ScriptBlock, $servers) {
    $arrayx = $servers | ForEach-Object { $_.servername + "|" + $_.ip }
    PSMultithreading $count $arrayx $ScriptBlock
    $Results | Sort-Object

}

function PrepIPandServername ($StringX) {
    $StringSplit = $StringX.Split('|')
    $global:ip = $StringSplit[1]
    $global:servername = $StringSplit[0]
 
}
```

然后是主要调用逻辑

```powershell
Import-Module C:\DELLipmiModule.psm1
servers -path    C:\cd03.csv -matchx "Wc|Sc"

#单条测试使用如下语句
#$ip="IP"
#$servername="机器名" 
#IPMICheckAndSet "get bios.BiosBootSettings.SetBootOrderEn"

#检查UEFI启动条目或者第一启动项目，注意pending
$ScriptBlock = {
    Param ([string]$StringX )
    Import-Module C:\DELLipmiModule.psm1
    PrepIPandServername $StringX
    IPMICheckAndSet 'get bios.BiosBootSettings.SetBootOrderEn'
    #IPMICheckAndSet "get bios.BiosBootSettings.UefiBootSeq"
}
DELLOutputInfomation 10 $ScriptBlock $servers


#开启HDD检测，UEFI引导，加一个计划任务，要求强制重启
$ScriptBlock = {
    Param ([string]$StringX )
    Import-Module C:\DELLipmiModule.psm1
    PrepIPandServername $StringX
    IPMICheckAndSet  'set  bios.BiosBootSettings.bootmode Uefi'
    IPMICheckAndSet  'set bios.BiosBootSettings.HddPlaceholder Enabled'
    IPMICheckAndSet  'jobqueue create BIOS.Setup.1-1 -s TIME_NOW -r forced'
}
DELLOutputInfomation 10 $ScriptBlock $servers
 

#修改启动顺序，特定磁盘在前面，并且不强制重启
$ScriptBlock = {
    Param ([string]$StringX )
    Import-Module C:\DELLipmiModule.psm1
    PrepIPandServername $StringX
    IPMICheckAndSet  “set bios.BiosBootSettings.SetBootOrderEn RAID.Slot.6-1”
    IPMICheckAndSet  “set bios.BiosBootSettings.SetBootOrderDis NIC.PxeDevice.1-1”
    IPMICheckAndSet  'jobqueue create BIOS.Setup.1-1 -s TIME_NOW -r none'
}
DELLOutputInfomation 10 $ScriptBlock $servers

 
#纯IPMI 重启机器
$servers  | ForEach-Object { C:\racadm\ipmitool.exe -I lanplus -H $_.ip -U root -P calvin chassis power reset }
 
```
