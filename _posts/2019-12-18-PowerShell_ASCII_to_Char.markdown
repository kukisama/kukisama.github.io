---
layout: post
title: "PowerShell从ASCII到字符的相互转换"
tags: PowerShell进阶学习
---

# 目标
目标是生成一个数组，生成`中华常用5000字`，或者`字母a到z`，`日语50音`之类的。

用途很多，比如大家来找茬，选出你曾经看到的字符，做验证码等等。

其实a到z还好说，但是常用字或者生僻字这种字典就很麻烦了。基于这些字符在计算机中都是`有序排列`的，只要想办法顺序输出就可以了。

# 原理
.net有方法可以对ASCII码和真实字符之间做转换，网上找了个例子，翻译成PowerShell
```powershell
## 从一个数组转换为一个字符
## 可以转换u3295 类似这种数据到真实的字符
# 字符编码转换1，输出是一个数组 
$enc = [system.Text.Encoding]::Unicode
$string1 = "鵢" 
$data1 = $enc.GetBytes($string1) 
$data1
# 从数组转换回字符
$enc = [System.Text.Encoding]::Unicode
$enc.GetString($data1)
 ```
 输出结果大概是这个样子的
 
 ![image](http://ny9s.com/picupdate/20191218102312.png)

# 可用代码
 ```powershell
 针对代码转换来转换区，写两个函数就可以了
   function GetCodePoint($ch)
    {
      #$retVal = "u+";
      $bytes = ([system.Text.Encoding]::Unicode).getbytes($ch.ToString())
      $global:bytex=$bytes
      for ($i = $bytes.length-1; $i -ge 0; $i--)
      { 
        $retVal += ($bytes[$i]).ToString("X2")  
      }
      $retVal
      #[char][int]("0x$retVal")
   }
  function GetTrueChar ($retVal)
    {[char][int]("0x$retVal")
}

 
#跑个例子
GetCodePoint 鵢
GetTrueChar 9D62
 ```
 

 
 看起来没问题，转换正常
 ![image](http://ny9s.com/picupdate/20191218102512.png)
 
```powershell
 #一些小贴士
 Characters
如何使用PowerShell将ASCII值转换为字符？
[char]64
如何使用PowerShell将字符转换为ASCII值？
[int][char]'@'
如何使用PowerShell生成英文字母？
[char[]](97..122)
```
# 测试

 写一个生成一大堆乱七八糟字符的函数看看效果，因为ASCII，类似9D62这种实际是16进制数，最后转换成10进制数，然后进行转换。
 
 所以我们基于```16进制```或者```直接10进制```进行累加就能出来一大堆需要的数
```powershell
 $STARTx=GetCodePoint "鵢"
 Write-Host "开始！"
1..100|%{
#16进制首先转换成10进制，然后增加偏移量$_，再转换回16进制(字符大写)
 Write-Host (GetTrueChar ([System.Convert]::ToInt32($STARTx,16)+$_).ToString('X'))  -NoNewline
 }
 ```
 
 ![image](http://ny9s.com/picupdate/20191218105159.png)
 
 
 用这种方法做出来的字典，非常`优雅`
 
这个写法还有一个变种，可以用类似下面的方法，直接简单生成。
```powershell
 $日文部分字库 = 0..50 | ForEach-Object { [char][int](0x306e + (0x01 * $_)) }
 ```
