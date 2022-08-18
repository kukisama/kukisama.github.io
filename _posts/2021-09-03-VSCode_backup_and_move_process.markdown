---
layout: post
title: "VSCode备份和搬家流程"
tags: PowerShell进阶学习
---

## 背景
在系统`重新安装`或需要将自己的配置环境`快速`的复制到其他主机上时，可以采用此方法

`多系统`环境下，可以通过重定向插件目录，免除在每个系统中都下载一次插件的问题

在`不联网`的环境中，可以使用此方法将插件提前下载好，一次性解压至目标主机

## 安装

前往 https://code.visualstudio.com/  根据操作系统下载安装即可



# 备份数据

总共需要备份两个内容，一个是插件，一个是主程序的安装目录

### 插件目录

VSCode的插件位于如下路径，将整个目录拷贝出来备用，如果调用的插件很多，则此处拷贝会比较耗时。

`C:\Users\ 你的用户名\.vscode\extensions`

### 安装目录

安装目录根据最开始安装时的定义进行选择，默认应该是

`C:\Program Files\VScode`

### 打包文件

使用自己常用的压缩工具，如`7zip`/`WinRAR`，将以上两个目录打包



# 还原数据
### 解压缩

根据需要，将`插件目录`和`安装目录`放在自己需要的路径即可。

### 修改映射
一般来说，经过日积月累的使用，插件目录的文件都会比较多，不建议拷贝回原始的`C:\Users\ 你的用户名\.vscode`路径，可以使用下面的方式，为插件目录增加一个`链接`，直接进行调用。

这种配置在多系统下也是很有必要的

```powershell
mklink  /j 需要链接的目录  真实目录
#举例
mklink  /j C:\Users\ 你的用户名\.vscode  c:\插件备份目录
mklink  /j C:\Users\kukisama\.vscode   D:\Programs\.vscode
```
具体的增删例子如下，[来源于微软](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/mklink)
```powershell
mklink /d \MyFolder \Users\User1\Documents
mklink /h \MyFile.file \User1\Documents\example.file
rd \MyFolder
del \MyFile.file
```




# 使用小技巧
### 切换中文
- 在VSCode的商店中，安装中文插件
- `Ctrl+Shift+P`快捷键 打开命令面板。
-  在命令面板中，输入Configure Display Language，选择Configure Display Language命令，切换语言

### PowerShell代码格式化

在拥有对应的插件的时候，可以用如下两个命令完成常用的操作。
另外需要注意，如果修改VSCode配置文件，可以做到在`刷文档的时候`，`自动替换别名`。

操作| 命令
---|---
格式化文档|Shift+Alt+F
替换别名 | Shift+Alt+E

###  一些插件
根据个人喜好安装即可，下面是我推荐的一些。
- Code Runner
- compareit 对比
- Excel Viewer 查看excel
- GitHub Pull Requests and Issues
- Inline Values support for PowerShell
- Powershell Extension Pack
- Prettier - Code formatter 代码格式化
- Regexp Explain 正则检查插件
- GIT
- vscode-icons 改变图标的颜色



