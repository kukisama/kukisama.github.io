---
layout: post
title: "从VMM获取主机带外地址"
tags: PowerShell进阶学习
---

## 需求
在使用SCVMM的时候，需要统计一下哪些主机配置了带外管理地址，以及使用了这个带外管理地址的主机名字是啥。
这其实是一个很简单的需求，问题只是在于这个带外的地址长的比较深`它藏在.PhysicalMachine.BMCAddress下`


```powershell
$date = date -Format MMddhhmmss
$a = Get-SCVMHost | Sort-Object -Property name
Write-Output "name,BMC" | Out-File ("c:\" + $date + "bmcaddress.csv") -Force
$a | ForEach-Object { Write-Host $_.name -NoNewline -ForegroundColor Red
    Write-Host $_.PhysicalMachine.BMCAddress -ForegroundColor Green
    Write-Output "==="
    $_.name + "," + $_.PhysicalMachine.BMCAddress | Out-File ("c:\" + $date + "bmcaddress.csv") -Force -Append
}
notepad  ("c:\" + $date + "bmcaddress.csv") 
```
