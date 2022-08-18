---
layout: post
title: "快速剔除不存在的Active Directory域信息"
tags: Active Directory域进阶学习
---

# 背景知识
在给大型的Active Directory环境做架构升级、调整、扩容、还原等操作时，（比如30节点以上的环境），直接拿生产练手属于作死的行为。所以搭建POC环境很有必要。

这篇文章是帮助大家解决在大型生产环境搭建Active Directory域POC时的一个繁琐步骤。

# 处理流程
按照传统做法，首先是备份环境中的任意一台Active Directory域控制器（最好是虚拟机），直接针对其导出快照镜像就可以了，然后在一个隔离网络的环境中进行测试就可以了。

这里问题就来了，因为是一个独立的环境，我们也不可能把几十台Active Directory都倒腾出来，所以需要在这台服务器上，依次删除其他几十个服务器的关联关系。当删除到只剩一台的时候，才能开始测试。

# 典型的处理一台主机的流程
正常流程大概是下面这样的，需要使用命令行来处理
```
C:\Windows\system32>ntdsutil
ntdsutil: metadata cleanup
metadata cleanup: select operation target
select operation target: connections
server connections:  connect to server dc01.contoso.com

绑定到 dc01.contoso.com ...
用本登录的用户的凭证连接 dc01.contoso.com。
server connections: quit

select operation target: list site
找到 1 站点
0 - CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=contoso,DC=com
select operation target: select site 0
站点 - CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=contoso,DC=com
没有当前域
没有当前服务器
当前的命名上下文
select operation target: list domains
找到 1 域
0 - DC=contoso,DC=com
select operation target: select domain 0
站点 - CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=contoso,DC=com
域 - DC=contoso,DC=com
没有当前服务器
当前的命名上下文
select operation target: list servers for domain in site

找到 4 服务器
0 - CN=DC01,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=contoso,DC=com
1 - CN=TDC03,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=contoso,DC=com
2 - CN=TDC04,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=contoso,DC=com
3 - CN=TDC05,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=contoso,DC=com
select operation target:此处准备复制脚本
```

# 问题来了
这里的问题是，我们需要一个一个的去选择服务器，然后删除，然后退到上一层，然后再操作。这真的是很麻烦

此时使用PowerShell生成一组命令，下方的1 .. 100 表示选择上面阶段找到的服务器，编号从1到100，这里需要根据实际需要选择，比如当前机器的编号是45，则脚本需要执行两次，一次是0-44，一次是46到100.
```powershell
cls
1 .. 100 | %{
	$x = "select server $_
quit
remove select server
select operation target"
	$x
}
```
脚本执行成功后，会在下方生成类似如下的文本字符，将文本字符拷贝出来，然后直接复制到上一阶段准备删除Active Directory环节的cmd中，系统就会连续执行，此时只需要鼠标点击确认即可。
 
之所以脚本会连续执行，是因为……有回车。
![image](http://ny9s.com/picupdate/20191125161725.png)

## 备注

如果节点正常，一般使用如下方法进行角色迁移，仅作记录。

```powershell
正常使用transfer
cmd
ntdsutil
roles
connections
connect to server "第二台DC的名稱"
quit
seize rid master
seize pdc
seize infrastructure master
seize domain naming master
seize schema master
```

