---
layout: post
title: "PowerShell批量查询域内主机磁盘剩余空间"
tags: PowerShell进阶学习
---

# 简单实现
- 几个需要关注的地方：
- 1、找出所有需要检查的主机名称，从Active Directory域中直接获取会很方便
- 2、确认能够访问的主机（ping），不能访问的不去检测磁盘。
- 3、针对能够访问的主机，检测本地磁盘可用空间和可用空间占比

```powershell
#找出xxxou下的所有主机
$AllComputers = Get-ADComputer -Filter * -SearchScope Subtree  -SearchBase "CN=Computers,DC=contoso,DC=com"
#定义一个函数，确认主机是否存活。
#这里使用了Write-Host，它只会在屏幕出现，不会被写入变量。可以方便排错。
function CheckDOA ($ComputerName) {
    $sum = ping $ComputerName -w 100 -n 1
    if ($sum -match "TTL") {
        $ComputerName
        Write-Host $ComputerName"可通信" -BackgroundColor Green  
    }
    else { Write-Host $ComputerName"不可通信，请检查" -BackgroundColor Red }
}
#定义一个检查磁盘空间情况的函数，这里用到了针对数字的格式化，{0:n1}为保留一位小数
function CheckCSpace ($ComputerName) {
    Invoke-Command $ComputerName { $data = Get-PSDrive C
        $env:COMPUTERNAME + ",剩余空间 " + [string]('{0:n1}' -f ($data.Free / 1GB)) + " GB" + ",剩余空间占比 " + ('{0:n1}' -f ($data.Free / ($data.Used + $data.Free) * 100 ) + "%" )
    }
}
#进行检查
$liveComputers = $AllComputers.name | ForEach-Object { CheckDOA $_ }
CheckCSpace $liveComputers


```
# 可供改进的方向
- 整个查询逻辑使用foreach，是顺序执行的，可以改为并发逻辑，批量执行。
- CheckCSpace部分使用的是icm，默认会有30线程，无需刻意修改。


```powershell


$Throttle = 100 #线程


$ScriptBlock = {
    Param (
        [string]$ComputerName
    )

    function CheckIP ($ComputerName) {
        $sum = ping $ComputerName -w 100 -n 1
        if ($sum -match "TTL") {
            $ComputerName
            Write-Host $ComputerName"可通信" -BackgroundColor Green  
        }
        else { Write-Host $ComputerName"不可通信，请检查" -BackgroundColor Red }
    }
    CheckIP $ComputerName
 
}
#创建一个资源池，指定多少个runspace可以同时执行

$RunspacePool = [RunspaceFactory]::CreateRunspacePool(1, $Throttle)
$RunspacePool.Open()
$Jobs = @()
 

 
$AllComputers.name | ForEach-Object {
    $Job = [powershell]::Create().AddScript($ScriptBlock).AddArgument($_)
    $Job.RunspacePool = $RunspacePool
    $Jobs += New-Object PSObject -Property @{
        Server = $_
        Pipe   = $Job
        Result = $Job.BeginInvoke()
    }
}
 

#循环输出等待的信息.... 直到所有的job都完成 
 
Write-Host "Waiting.." -NoNewline
Do {
    Write-Host "." -NoNewline
    Start-Sleep -Seconds 1
} While ( $Jobs.Result.IsCompleted -contains $false)
Write-Host "作业已完成" -BackgroundColor Green


#输出结果 
$Results = @()
ForEach ($Job in $Jobs) {
    $Results += $Job.Pipe.EndInvoke($Job.Result)
}

CheckCSpace $Results

```

# 完整代码，逻辑优化
- 针对并发逻辑，单独包成函数
- 不仅仅可以用来ping

```powershell

$AllComputers = Get-ADComputer -Filter * -SearchScope Subtree  -SearchBase "CN=Computers,DC=contoso,DC=com"
#region 函数与脚本块
$ScriptBlock = {
    Param (
        [string]$ComputerName
    )
    function CheckIP ($ComputerName) {
        $sum = ping $ComputerName -w 100 -n 1
        if ($sum -match "TTL") {
            $ComputerName
            Write-Host $ComputerName"可通信" -BackgroundColor Green  
        }
        else { Write-Host $ComputerName"不可通信，请检查" -BackgroundColor Red }
    }
    CheckIP $ComputerName
 
}
#创建一个资源池，指定多少个runspace可以同时执行

function CheckCSpace ($ComputerName) {
    Invoke-Command $ComputerName { $data = Get-PSDrive C
        $env:COMPUTERNAME + ",剩余空间 " + [string]('{0:n1}' -f ($data.Free / 1GB)) + " GB" + ",剩余空间占比 " + ('{0:n1}' -f ($data.Free / ($data.Used + $data.Free) * 100 ) + "%" )
    }
}

function PSMultithreading ($Throttle, $allnodename, $ScriptBlock) { #线程，需要操作的对象，脚本块
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
#endregion 函数与脚本块
PSMultithreading 100 $AllComputers.name $ScriptBlock
CheckCSpace $Results
```

# 遗留问题
- 针对ping不通的，如何进行展示。是忽略还是以其他方式表现。
- 使用并发逻辑之后，实际上可以在脚本块中同时完成检测和检查两个步骤。
