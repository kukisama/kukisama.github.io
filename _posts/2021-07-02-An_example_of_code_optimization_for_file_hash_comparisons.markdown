---
layout: post
title: "文件哈希比较的代码优化一例"
tags: PowerShell优化
---


# 背景

某个场景，需要手动对业务系统替换几个文件，文件位于不同的路径下，且文件量大。

在替换结束后，需要检查文件是否真的被替换了。是否被替换了合适的版本。

## 实现

检查文件有没有货不对板有多种方式。如果是exe这类正经封装过的文件，一般会有FileVersionRaw属性，属性中会有版本号，我们基于此比较就可以。但如果是脚本、文本文件这种没有版本号的，这个方法则不适用了。另外版本号也有可能会骗人，版本起名全靠作者自觉。

基于此，可以使用PowerShell内置命令[Get-FileHASH](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash?view=powershell-7.1)来比对每一个文件的哈希值，通过比对哈希值（默认SHA256），来判断文件是否被替换成正确的版本。

```powershell
PS C:\windows\system32> Get-FileHash $a[888]

Algorithm       Hash                                                                   Path                                                                  
---------       ----                                                                   ----                                                                  
SHA256          B93A3F3DC478B4D167F26BCDEB4E4984DA997665AC79B91BCA3F1DF4E36A7E5E       C:\windows\system32\dafWCN.dll 
```

Get-FileHASH除了支持使用SHA256来计算外，也支持如下格式

- SHA1
- SHA256
- SHA384
- SHA512
- MD5

# 代码实现

首先准备两个函数，一个用来抓数据

```powershell
function OutputHASHCheckFile ($Filename, $CheckPath,$OutputPath) {
  $CheckPath=$CheckPath.replace(':','$')
  $FullFile = Get-ChildItem $("\\" + $Filename + "\" + $CheckPath) -Recurse
  $Files = $FullFile | Where-Object { $_.Attributes -eq "Archive" }
  $FileHash = $Files | ForEach-Object { Get-FileHASH $_.FullName }
  "HASH,path,FileVersion" | Out-File $($OutputPath + $Filename + ".csv") -Encoding utf8  
  $FileHash | ForEach-Object { $_.HASH + "," + $_.path.split('$')[-1] + "," + (Get-Item ($_.path)).VersionInfo.FileVersionRaw } | Out-File $($OutputPath + $Filename + ".csv") -Encoding utf8 -Append
}
```

这里有一个大问题，就是类似```C:\windows\system32\dafWCN.dll ``` 这种路径格式，当包含```\```符号时，PowerShell的解析是有问题的，这个时候不能用```match```,但是可以用`EQ` ,非常之神奇。

```powershell
\Windows\System32\zh-CN\acledit.dll.mui
PS C:\windows\system32> 
PS C:\windows\system32> $Basefile.path
\Windows\System32\zh-CN\aadtb.dll.mui
\Windows\System32\zh-CN\aadWamExtension.dll.mui
\Windows\System32\zh-CN\AboutSettingsHandlers.dll.mui
...........
PS C:\windows\system32>   $Basefile.path -match $Targetfile[4].path
#不同操作系统的返回不太一样，但都是返回错误
PS C:\windows\system32>   $Basefile.path -eq $Targetfile[4].path
\Windows\System32\zh-CN\acledit.dll.mui
```

再做一个函数用来比对哈希值，这里用到了相对很复杂的例子，首先从目标目录中找出和基线目录中一致的```目录```（也就是挑出来），然后从整个目标目录中，找出来这条完整的```数据```（包含目录、哈希值、版本号）

```powershell
$Targetfile | Where-Object {$_.path -eq ($Targetfile.path -eq $temp.path) }
```

完整代码如下

```powershell
function HASHCheckFile ($baselineFile, $targetFile) {
  Write-Host $_ -ForegroundColor Green
  $Basefile = Import-Csv  $baselineFile  
  $Targetfile = Import-Csv  $targetFile 
  $Basefile | ForEach-Object {
    $temp=$_
    $identicalfile = $Targetfile | Where-Object {$_.path -eq ($Targetfile.path -eq $temp.path) }
    if ($identicalfile.HASH -ne $_.HASH) {
      Write-Host ($_.path + " | " + $_.Fileversion) -ForegroundColor Red
    }
  }   
}
```

# 优化1

上面代码是可以成功的,但是问题是速度太慢了。我插入了一条获取当前时间的逻辑，可以看到每一次比对文件都需要消耗1秒多，对于有1700多个文件的目录而言，就是需要消耗1700多秒。显然是无法接受的。

```powershell
function HASHCheckFile ($baselineFile, $targetFile) {
  Write-Host $_ -ForegroundColor Green
  $Basefile = (Import-Csv  $baselineFile) 
  $Targetfile = (Import-Csv  $targetFile) 
  $Basefile | ForEach-Object {
    $temp=$_
    $identicalfile = $Targetfile | Where-Object {$_.path -eq ($Targetfile.path -eq $temp.path) }
    (Get-Date).TimeOfDay.TotalSeconds #检查时间信息
    if ($identicalfile.HASH -ne $_.HASH) {
      Write-Host ($_.path + " | " + $_.Fileversion) -ForegroundColor Red
    }
  }   
}
#可以看到输出结果，每一次查询，会消耗1秒多
64498.099711
64499.1767082
64500.2567158
```

为什么会这么慢的？这完全是查询方式的锅。

```powershell
$Targetfile | Where-Object {$_.path -eq ($Targetfile.path -eq $temp.path) }
```

那么这里面是谁慢导致的呢？我们看下面的例子。当我们单纯用`EQ`的方式来匹配数据的时候，只用了2ms，但是换成管道，然后在右侧使用`Where-Object`来进行查询呢？达到了惊人的`90ms`，这就差了几十倍。

```powershell
$a=1..10000

PS C:\windows\system32> (Measure-Command {$a -eq 22}).TotalMilliseconds

2.1583

PS C:\windows\system32> (Measure-Command { $a | Where-Object { $_ -eq 22 } }).TotalMilliseconds

90.061
```

如果我们换成`foreach`这种循环来做呢？只需要22ms。

```powershell
(Measure-Command { foreach ($item in $a) {
      if ($item -eq 22) {
        Write-Host $item 
      }
    } }).TotalMilliseconds
22
22.3433
```

解题思路就是尽量不要在这里用管道，实在需要遍历的话，用foreach也可以。由于路径格式，包含```\```符号。所以至少需要使用一次循环，这里先尝试下用foreach

```powershell
#脚本1
function HASHCheckFile ($baselineFile, $targetFile) {
  Write-Host $_ -ForegroundColor Green
  $Basefile = (Import-Csv  $baselineFile) 
  $Targetfile = (Import-Csv  $targetFile) 
  $Basefile | ForEach-Object {
    $temp = $_
    $identicalfile = foreach ($item in  $Targetfile ) {
      if ($item.path -eq ($Targetfile.path -eq $temp.path))
      {
        $item
      }
    }
     # $Targetfile | Where-Object {$_.path -eq ($Targetfile.path -eq $temp.path) }
    (Get-Date).TimeOfDay.TotalSeconds #检查时间信息
    if ($identicalfile.HASH -ne $_.HASH) {
      Write-Host ($_.path + " | " + $_.Fileversion) -ForegroundColor Red
    }
  }   
}
```

试试看运行效果

```powershell
PS C:\windows\system32> HASHCheckFile C:\localhost.csv C:\localhostold.csv

66598.8235475
66599.847084
66600.8921025
66601.9200949
```

看起来还是没什么改进。

# 优化2

仔细观察逻辑

```powershell
$Basefile | ForEach-Object {
    $temp = $_
    $identicalfile = foreach ($item in  $Targetfile ) {
      if ($item.path -eq ($Targetfile.path -eq $temp.path))
      {
        $item
      }
    }
```

会发现，`  if ($item.path -eq ($Targetfile.path -eq $temp.path))`中的`$Targetfile.path -eq $temp.path`无论再简单，运算再快，还是无可避免的在循环中计算了一次。我们把它存储在变量中，重复调用一下

```powershell
#脚本2
function HASHCheckFile ($baselineFile, $targetFile) {
  Write-Host $_ -ForegroundColor Green
  $Basefile = (Import-Csv  $baselineFile) 
  $Targetfile = (Import-Csv  $targetFile) 
  $Basefile | ForEach-Object {
    $temp = $_
    $temppath = $Targetfile.path -eq $temp.path
     $identicalfile = foreach ($item in  $Targetfile ) {
      if ($item.path -eq $temppath) {
        $item
      }
    }
    (Get-Date).TimeOfDay.TotalSeconds #检查时间信息
    if ($identicalfile.HASH -ne $_.HASH) {
      Write-Host ($_.path + " | " + $_.Fileversion) -ForegroundColor Red
    }
  }   
}
```
肉眼可见的速度飞快起来，执行一条大约需要0.01多一点秒（10ms）
```powershell
PS C:\windows\system32> HASHCheckFile C:\localhost.csv C:\localhostold.csv
66844.7875269
66844.8005286
66844.8115265
```

再把逻辑里面的管道换成`foreach`

```powershell
#脚本3
function HASHCheckFile ($baselineFile, $targetFile) {
  Write-Host $_ -ForegroundColor Green
  $Basefile = (Import-Csv  $baselineFile) 
  $Targetfile = (Import-Csv  $targetFile) 
  foreach ($itemnew in   $Basefile) {
    $temp = $itemnew
    $temppath = $Targetfile.path -eq $temp.path
    $identicalfile = foreach ($item in  $Targetfile ) {
      if ($item.path -eq $temppath) {
        $item
      }
    }
    (Get-Date).TimeOfDay.TotalSeconds #检查时间信息
    if ($identicalfile.HASH -ne $temp.HASH) {
      Write-Host ($temp.path + " | " + $temp.Fileversion) -ForegroundColor Red
    }
  }   
 
 
}
```

看起来接近0.006秒了（6ms）

```powershell
PS C:\windows\system32> HASHCheckFile C:\localhost.csv C:\localhostold.csv

67540.2396194
67540.2476195
67540.2526189
```

可能是样本量太少，我将脚本完整跑一次得出结果。

| 脚本  | 执行时间                     |
| ----- | ---------------------------- |
| 脚本2 | 14.0271825                   |
| 脚本3 | 10.4654441秒                 |
| 脚本1 | 单条执行速度太慢，不参与评比 |

# 优化3

10秒跑一个1700多文件的对比，基本上可以算是符合要求了，生产中用起来也没问题。不过有没有更快的呢？由于我用的是`import-csv`的方式加载的对象，每个对象包含了路径、哈希值以及版本号。

但是对我的需求而言，如果要检查某个文件的`哈希值是否匹配`，那么他在(基准文件)中的记录，一定有一条`完全一模一样`的数据同时存在于（待查询文件）中，但凡差一个字，都不是真的一样。

基于这个原理，只需要查询一次，不需要先查路径再查哈希值。也就是循环只需要做一次。所以导入数据的时候用`GC`进行原样导入，不转换成对象。

```powershell
#代码4
function HASHCheckFile ($baselineFile, $targetFile) {
  Write-Host $_ -ForegroundColor Green
  $Basefile = Get-Content   $baselineFile 
  $Targetfile = Get-Content   $targetFile 
  foreach ($item in   $Basefile) {
    $temppath = $Targetfile -eq $$item 
    if ([string]$temppath -eq "") {
      Write-Host $$item -ForegroundColor Red
    }
  }   
}
```
看看消耗时间，达到了惊人的0.48秒，把其他方式按在地上摩擦。
```powershell
0E259FD07FF6D8D38BB529724E965A11037AF4ED260EEF3CB3692B53B8D3BDF4,\Windows\System32\zh-CN\HVDirectAD.ps1,0.0.0.0
F32626EE7F8FF0CB4A7CB5D5026E88F7F9857780CEA1D613B73D722375F8BCEF,\Windows\System32\zh-CN\PS代码格式化临时.ps1,0.0.0.0
0.4824708
```

当然这种输出格式需要一点点美化，不过和效率比起来也没什么了。

# 总结

- 不要反复的计算某一件事情，如果计算结果阶段一致，就一定要把它放在变量里。
- 用` foreach`替换管道，可以明显提升效率。需要注意，`foreach`和`foreach-object`是完全不同的两个东西
- 用蠢萌蠢萌的字符串处理，只要能保证循环的次数少，那效率就是最高的。

