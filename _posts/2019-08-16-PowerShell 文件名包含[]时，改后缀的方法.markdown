# 需求
众所周知的问题，现在视频网站发布视频的方式方法已经有了很大的改变。比如mp4文件改名为mp41，类似下面这样
```powershell
PS E:\BaiduNetdiskDownload> ls *.mp41 -Recurse

目?录?: E:\BaiduNetdiskDownload

Mode                LastWriteTime         Length Name                                                                    
----                -------------         ------ ----                                                                    
-a----        2019/8/11     20:42     1322066936 [SUBPIG][Eien no Nispa SP].mp41
```

然而实际按照

```powershell
[SUBPIG][Eien no Nispa SP].mp41
```
这个文件名，用rename的方式去修改，却被提示失败。


```powershell
PS E:\BaiduNetdiskDownload> Rename-Computer "[SUBPIG][Eien no Nispa SP].mp41" "[SUBPIG][Eien no Nispa SP].mp4"
Rename-Computer : 找不到接受实际参数“[SUBPIG][Eien no Nispa SP].mp4”的位置形式参数。
所在位置 行:1 字符: 1
+ Rename-Computer "[SUBPIG][Eien no Nispa SP].mp41" "[SUBPIG][Eien no N ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidArgument: (:) [Rename-Computer]，ParameterBindingException
    + FullyQualifiedErrorId : PositionalParameterNotFound,Microsoft.PowerShell.Commands.RenameComputerCommand
```

# 自动补齐
这是为什么呢？其实原因很简单，因为PowerShell中，对于[]符号有特殊的定义，所以不能这么写。
我们试试在命令行下直接运行它，系统会自动补齐什么样的字符转义

```powershell
PS E:\BaiduNetdiskDownload> & '.\[SUBPIG][Eien no Nispa SP].mp41'
```

可以看到字符串前面有&  然后跟一个空格，字符串被' （单引号）包裹，这种方法可以用来运行一些包含了特殊字符的命令行工具，或者可执行程序，但是，在传递参数的时候还是用不了的。

我们试试，在某一个参数中，指定路径或者文件名，看系统自动补齐会变成什么样子


```powershell
PS E:\BaiduNetdiskDownload> Rename-Item -Path '.\`[SUBPIG`]`[Eien no Nispa SP`].mp41'
```

大概这就是正确方式了吧，专门写了个函数

```powershell
function EscapeWord ($param1) {
    if ($param1 -match "[[]" -or $param1 -match "[]]") {
        $param1.replace('[', '`[').replace(']', '`]')
    }
}
EscapeWord "E:\BaiduNetdiskDownload\[SUBPIG][FAKE AFFAIR EP05].mp41"
E:\BaiduNetdiskDownload\`[SUBPIG`]`[FAKE AFFAIR EP05`].mp41
```

先单独一条数据执行下

```powershell
PS E:\BaiduNetdiskDownload> Rename-Item -Path '.\`[SUBPIG`]`[Eien no Nispa SP`].mp41' -NewName '.\`[SUBPIG`]`[Eien no Nispa SP`].mp4' 
Rename-Item : 指定路径 E:\BaiduNetdiskDownload\`[SUBPIG`]`[Eien no Nispa SP`].mp41 下的对象不存在。
```

实际来看，还是失败了。

# 变通解决
实在是挺无语的，既然[]在PowerShell下被重新定义了，那CMD下总是好的吧。用CMD来混合一下。这次的做法是切割名称字符串，把最后一个字符串切掉，然后替换名称。依然可以达到需求，并且修改记录还会追加保存在一个文件中。

```powershell
(Get-ChildItem *.mp41 -Recurse) | ForEach-Object { $fullname = $_.fullname 
    $fullname
    $NewName = $_.name
    $NewName = ($NewName[0..($NewName.Length - 2)] -join "")
    $NewName
    $info = "rename `"$fullname`" `"$NewName`""
    cmd /c $info
    $info | Out-File c:\333.txt -Append -Force -Encoding utf8
}
```


# 最终方案
问题是解决了，但是实现方法太拧巴。如果有原生方法，上面的操作还是不会考虑的。
我通过搜关键字
>rename file PowerShell match [] 

在东家找到了下面一个最终极的方案。
https://answers.microsoft.com/en-us/windows/forum/windows_10-files/how-to-rename-image-files-in-a-folder-all-to-jpg/2a7e2873-e04b-472b-b239-afad2f2020fc

果然解决不了的时候直接上.net都是一件利器。

```powershell
Get-ChildItem *.mp41 -Recurse | Rename-Item -newname { [io.path]::ChangeExtension($_.name, "mp4") }
```


在https://stackoverflow.com/questions/5574648/use-regex-powershell-to-rename-files 也发现一个很有意思的替换方法，不过一样的，受限于[]，不太适合我的场景。

```powershell
Get-ChildItem *.mp41 | ForEach-Object{ Rename-Item $_ $(($_.name -replace '^filename_+','') -replace '_+',' ') }
```


# 剩下的问题
其实到这里，还有一个坑，如果有一天，需要对包含[]的文件，替换文件名，应该如何做呢？

# 完美解决
感谢`唐老师在微信上指点`，在
https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/rename-item?view=powershell-6
找到了这么一个参数。可以对路径名称不进行转义。经过测试，完美解决
```powershell
-LiteralPath
Specifies a path to one or more locations. The value of LiteralPath is used exactly as it is typed. No characters are interpreted as wildcards. If the path includes escape characters, enclose it in single quotation marks. Single quotation marks tell PowerShell not to interpret any characters as escape sequences.
。
PS E:\BaiduNetdiskDownload> $a =Get-ChildItem *.mp41 -Recurse
PS E:\BaiduNetdiskDownload> Rename-Item -LiteralPath $a.FullName -NewName "[SUBPIG][FAKE AFFAIR EP05].mp4"

```
