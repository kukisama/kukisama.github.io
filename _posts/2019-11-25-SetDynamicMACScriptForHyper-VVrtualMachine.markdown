---
layout: post
title: "为Hyper-V虚拟机设置动态MAC的脚本"
tags: PowerShell进阶学习
---
# 需求
问题是这样的，手里有一波虚拟机，已经有MAC地址了，而且是静态的，但是某一天需要给他搬家，虚拟机所在的新家（VMM）限制了MAC地址段。所以需要给这些虚拟机先设置成动态MAC，让他们拿到新地址，然后再把新地址固化下来。

# 实现
手动操作就不说了，依然代码实现。老实说，我找了很久的代码。基本都是实现反向需求，就是给虚拟机设置静态MAC。但是我这不一样，我想设置成动态的啊。

首先感谢[这位大哥](#https://blogs.msdn.microsoft.com/taylorb/2013/08/12/changing-the-mac-address-of-nic-using-the-hyper-v-wmi-v2-namespace/
)，大哥用了很传统的WMI开干，解决了问题。

很可惜，PowerShell没有原生命令来解决这个问题
```powershell

#给虚拟机设置动态MAC的需求
#只是针对单网卡操作，如果一个机器有多个网卡，逻辑不匹配
function SetVMmacDyna ($vmName)
{
   #Retrieve the Hyper-V Management Service, ComputerSystem class for the VM and the VM’s SettingData class. 
$Msvm_VirtualSystemManagementService = Get-WmiObject -Namespace root\virtualization\v2 `
      -Class Msvm_VirtualSystemManagementService 

$Msvm_ComputerSystem = Get-WmiObject -Namespace root\virtualization\v2 `
      -Class Msvm_ComputerSystem -Filter "ElementName='$vmName'" 

$Msvm_VirtualSystemSettingData = ($Msvm_ComputerSystem.GetRelated("Msvm_VirtualSystemSettingData", `
     "Msvm_SettingsDefineState", `
      $null, `
      $null, ` 
     "SettingData", `
     "ManagedElement", `
      $false, $null) | ForEach-Object {$_}) 


#Retrieve the NetworkAdapterPortSettings Associated to the VM. 
#注意这里，中文系统和英文系统对网卡的命名是不一样的

$Msvm_SyntheticEthernetPortSettingData = ($Msvm_VirtualSystemSettingData.GetRelated(` 
     "Msvm_SyntheticEthernetPortSettingData") `
      | Where-Object {$_.ElementName -eq "网络适配器"}) 
      #网络适配器
      #Network Adapter
#Set the Static Mac Address To False and the Address to an Empty String
$Msvm_SyntheticEthernetPortSettingData.StaticMacAddress = $false
$Msvm_SyntheticEthernetPortSettingData.Address = ""

$Msvm_VirtualSystemManagementService.ModifyResourceSettings($Msvm_SyntheticEthernetPortSettingData.GetText(2))

  
}
#一把梭，设置完成
(get-vm|Where-Object{$_.State -eq "off"}).name|ForEach-Object{
SetVMmacDyna $_}
```
