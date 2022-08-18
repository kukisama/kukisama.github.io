# 症状
因为将一组群集，多次在VMM中进行添加删除的维护。多次操作之后，我发现群集在VMM中无法创建SOFS的共享卷，其他表现正常。 
### 错误0

> 错误(26561)
在存储池 S2D on HFPD01STOR01 上创建卷 Share576855D2 失败，错误代码为 4 [SM_RC_GENERIC_FAILURE]。Cannot convert 'System.Object[]' to the type 'Microsoft.Management.Infrastructure.CimInstance' required by parameter 'InputObject'. Specified method is not supported.

> 建议的操作
请指定用于创建卷的有效参数。


### 错误1
> 错误(12700)
由于以下错误，VMM 无法在“主机名”服务器上完成主机操作: Storage migration for virtual machine 'fghyju8989' (EC699F08-CCBE-45C0-B7E8-DB4D319E7A29) failed with error 'General access denied error' (0x80070005).

> Failed to set security info for '\\主机名\VMSTOR14\XXXX\fghyju8989': 'General access denied error'('0x80070005').
Unknown error (0x8001)

> 建议的操作
请解决此主机问题，然后重试该操作

针对错误0，怀疑'Microsoft.Management.Infrastructure.CimInstance'这句是根本原因。但是经过很多尝试，包括重建群集，重建S2D，依然无法解决这个问题。
因此转向手动创建所需的SOFS共享，并由VMM接管

# 处理流程
处理过程使用如下命令
```powershell
#创建一个到目标群集的cismsession
$newsession=New-CimSession 目标群集名称
#在群集新创建一个3路镜像卷，
New-Volume -CimSession $newsession  -FriendlyName "test02" `
-FileSystem CSVFS_ReFS -StoragePoolFriendlyName S2D* -Size 3GB -ResiliencySettingName Mirror 

#针对群集配置权限
New-SmbShare -CimSession $newsession  -Name "test02" `
-Path "C:\ClusterStorage\test02\"  -FullAccess "域名\账号", "域名\计算机名$"

```
到这里出现了新的问题，创建的目录可以在VMM中刷新出来，也可以使用如下代码，在VMM中接管
```powershell
#FS1所有共享
$FileShare = Get-SCStorageFileShare  |?{$_.StorageFileServer -match "SOFS名称"}
#存储文件服务器 
$FileServer = Get-SCStorageFileServer "SOFS的FQDN"
Set-SCStorageFileServer -StorageFileServer $FileServer   -AddStorageFileShareToManagement $FileShare 

```
但是，目录只能正常配置"共享权限",不能配置"安全权限",而且最严重的是，创建出来的共享目录，虚拟机不能迁移，例如在群集1的磁盘1到磁盘2迁移，或者群集1到群集2迁移。基本属于不可用状态。

输入常规的配置命令，系统会显示配置成功，但实际不会生效。例如

```powershell
Grant-SmbShareAccess -CimSession $newsession -Name $_ -AccountName "$($env:USERDOMAIN)\Storage Server Admins" -AccessRight Full -Confirm:$false 
Grant-FileShareAccess -CimSession $newsession -Name $_ -AccountName "$($env:USERDOMAIN)\common share users" -AccessRight Full -Confirm:$false 

```
以及

```powershell
get-acl \\群集服务器地址\VMSTOR12 | fl
$aclobj=get-acl  \\群集服务器地址\VMSTOR14 
 set-acl \\需要配置的群集服务器地址\VMSTOR111 -AclObject $aclobj

```
还有

```powershell
takeown /F \\群集服务器地址\VMSTOR04 /A /R /D Y
takeown /f * /a /r /d y

```
这些都不行，效果都一样。
```powershell
icacls \\群集服务器地址\VMSTOR04 /setowner "域名\omadmin" #/T /C
```
测试了两天，还是没有找到原因。

# 解决
经过仔细检查，发现通过SOFS创建的目录，大致是这样的

> C:\ClusterStorage\Share0BAD227D\Shares\VMSTOR01

>目录结构：存储空间→创建一个随机名的卷Share0BAD227D，在卷下创建Shares目录，在Shares目录下创建VMSTOR01

而自己手动用命令创建的是这样的
> C:\ClusterStorage\VMSTOR01> 

> 目录结构：存储空间→创建一个固定命名的卷VMSTOR01

这两者的区别在于，VMM的做法是在卷上创建了文件夹，基于文件夹定义权限。
所以目标是改成下面这种形式，测试一下。
> C:\ClusterStorage\VMSTOR01\share> 

关键代码如下
```powershell
New-Item  C:\ClusterStorage\$xname\share -ItemType Directory
New-SmbShare -CimSession $newsession  -Name $xname -Path "C:\ClusterStorage\$xname\share" -FullAccess  '域名\domain admins','administrators'

```
# 结论

关键点在于这里，不能将SOF创建的volume直接作为共享目录，需要在这个volume上创建目录，然后针对目录操作就可以了。这有点类似于不能针对一个磁盘操作，但可以针对磁盘上的目录操作。
