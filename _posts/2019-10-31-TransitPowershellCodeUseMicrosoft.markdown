---
layout: post
title: "如何更智能的看懂PowerShell的英文注释"
tags: PowerShell进阶学习
---

# 引言
无论是什么语言的代码，合理的注释以及手册都是交付的必要条件。
经过[上一个章节](http://github.ny9s.com/2019/10/%E5%A6%82%E4%BD%95%E7%BC%96%E5%86%99%E5%AE%8C%E5%85%A8%E6%97%A0%E6%B3%95%E7%BB%B4%E6%8A%A4%E7%9A%84PowerShell%E8%84%9A%E6%9C%AC/)，相信大家可以明白`给变量命名的重要性。`现在新的问题来了：

>  存在语言障碍的情况下，怎么去读懂别人的代码，或者说，更明白的看懂注释。

# 学外语
我从不掩盖自己屎一样的外语。从多年工作来说，外语薄弱会有很多影响，`但对编程的影响不是致命的。`
# 翻译
现在机器翻译已经做到相对足够好，比如下面[这家公司](http://www.microsoft.com)的产品

> [Microsoft 文本翻译 API](https://azure.microsoft.com/zh-cn/pricing/details/cognitive-services/translator-text-api/) 是一项基于云的机器翻译服务，支持多种语言，其支持的语言覆盖全球国内生产总值 (GDP) 95% 以上的区域。使用 Translator 可构建应用程序、网站、工具或任何需要多语言支持的解决方案。

它提供有免费的API接口，每个月可以免费用200万字。更贵的套餐也有，比如我现在用的 ` S1套餐，$40/百万 个字符的自定义翻译`

# API
在使用它之前，需要先注册一个账号，具体的操作方法可以在[官方网站的快速入门](https://docs.microsoft.com/zh-cn/azure/cognitive-services/translator/quickstart-translate?pivots=programming-language-go/)查找，很简单。
接下来我们需要看一下[他的API](https://docs.microsoft.com/zh-cn/azure/cognitive-services/translator/reference/v3-0-reference/)

比较可惜也是情理之中的事情是，并没有PowerShell相关的范例代码。根据[这个范例](https://docs.microsoft.com/zh-cn/azure/cognitive-services/translator/reference/v3-0-translate)，以及C#大哥的参考。
下面来构造PowerShell（.net家族小弟）的查询代码
> 首先是两个核心函数，通过这个函数，朋友们可以学习下关于WEB请求部分，C#代码是如何翻译成PowerShell。
- 获取所有支持的语言版本
- 翻译

```powershell

function GetAllTranslatorLang {
	((Invoke-WebRequest https://api.cognitive.microsofttranslator.com/languages?api-version=3.0).Content | ConvertFrom-Json).translation
}

function translatorMS ($language, $texts) {
	$APIuri = "https://api.cognitive.microsofttranslator.com/translate?api-version=3.0&to=$language" #&from=zh-Hans
	$translatordata = Invoke-WebRequest $APIuri ` -Body $texts -ContentType "application/json;charset=utf-8" -Method POST -Headers $postParams
	($translatordata.Content | ConvertFrom-Json).translations.text
}


```

- 注意用到的KEY，这个要提前申请，

```powershell
$subscriptionKey = '这里是key'
$postParams = @{ "Ocp-Apim-Subscription-Key" = "$subscriptionKey" }
```

```powershell
#然后这么使用函数就可了
translatorMS -language $lanx -texts $body
```

是不是~~很简单~~？通过上满几行，我们举一反三，就可以写出如下完整代码.

# ~~用法也很简单~~
1. 把下面的代码里面的`KEY`换成自己申请的，
2. 然后代码存储为.ps1
3. 想办法把它转换成exe
4. 最后将需要翻译的ps1文件，拖动到这个文件上,默认情况下支持中英互转
5. 如果你不在步骤2转换为exe，则需要使用类似如下的方法来使用

```powershell
 & '.\Bing Transit 0.1.ps1' -filename C:\x\azaa.ps1 -languages "zh-Hans"
```
> 最终结果：自动生成一个同名目录，里面有翻译好注释的ps1文件，以及所有抽出来的注释代码

```powershell

param ($filename,
	$languages = @("zh-Hans", "en"))

#param ($filename,$languages = @("zh-Hans", "ja", "en", "ko"))
#region function
function CreateNewFolder ($NewFolder) {
	if (!(Test-Path $NewFolder)) {
		mkdir $NewFolder | Out-Null
	}
}

function GetAllTranslatorLang {
	((Invoke-WebRequest https://api.cognitive.microsofttranslator.com/languages?api-version=3.0).Content | ConvertFrom-Json).translation
}

function translatorMS ($language, $texts) {
	$APIuri = "https://api.cognitive.microsofttranslator.com/translate?api-version=3.0&to=$language" #&from=zh-Hans
	$translatordata = Invoke-WebRequest $APIuri ` -Body $texts -ContentType "application/json;charset=utf-8" -Method POST -Headers $postParams
	($translatordata.Content | ConvertFrom-Json).translations.text
}

function ConvertLanguage ($comments) {
	for ($i = 0; $i -lt $comments.count; $i++) {
		$x = $comments[$i].split(',')
		if ($x[1] -match "region" -or $x[1] -match "\A#>" -or $x[1] -match "=" -or $x[1] -match "\$") {
			$comments[$i]
		}
		else { $x[0] + ",#" + $translatored[$i] }
	}
}


#endregion function


#$filename = "C:\x\ConvergedDeploy-V1.1.ps1"
$subscriptionKey = '这里是KEY'
$postParams = @{ "Ocp-Apim-Subscription-Key" = "$subscriptionKey" }
$gcfile = Get-Content $filename -Encoding UTF8
$info = New-Object System.Collections.ArrayList
$info2 = New-Object System.Collections.ArrayList
$fileNewname = $filename.Split('\')[-1].Substring(0, $filename.Split('\')[-1].Length - 4)
$NewFolder = ($filename.Split('\')[0 .. ($filename.Split('\').count - 2)] -join "\") + "\" + $fileNewname + "\"
CreateNewFolder $NewFolder
#创建原始文件、提取的描述文件、创建删除所有注释的文件
for ($i = 0; $i -lt $gcfile.count; $i++) {
	if ($gcfile[$i].Trim() -match "\A#.*") {
		$info2.Add([string]$i + "," + $gcfile[$i].Trim()) | Out-Null
	}
	elseif ($gcfile[$i].Trim() -eq "") { } #删除所有空行，可选
	else {
		$info.Add($gcfile[$i]) | Out-Null
	}
}
$gcfile | Set-Content $NewFolder$fileNewname".orgin.ps1" -Force -Encoding utf8
$info | Set-Content $NewFolder$fileNewname".Production.ps1" -Force -Encoding utf8
$info2 | Set-Content $NewFolder$fileNewname".comments.txt" -Force -Encoding utf8

############翻译，根据comments写回注释
$filename = $NewFolder + $fileNewname + ".orgin.ps1"
$fileNewname = $filename.Split('\')[-1].Substring(0, $filename.Split('\')[-1].Length - 10)
$orginfile = Get-Content $filename -Encoding UTF8

#获取所有支持的语言GetAllTranslatorLang
Get-ChildItem $NewFolder *.transedcomments.* | Remove-Item
#获取原始文件的注释
$comments = Get-Content $NewFolder$fileNewname".comments.txt" -Encoding utf8
#50一组，分割一下
$commentslengthcount = [math]::ceiling($comments.count / 50)

for ($i2 = 1; $i2 -le $commentslengthcount; $i2++) {
	if ($i2 -ne $commentslengthcount) {
		$ctx = $comments[(0 + $i2 * 50 - 50) .. ($i2 * 50 - 1)]
	}
	else {
		$ctx = $comments[(0 + $i2 * 50 - 50) .. ($comments.Count - 1)]
	}
	
	$body = $ctx | ForEach-Object { $_ | Select-Object @{ name = 'text'; e = { $_.split(',')[-1].replace('#', '') } } } | ConvertTo-Json
	
	$languages | ForEach-Object {
		$lanx = $_
		$translatored = translatorMS -language $lanx -texts $body
		$commentsnew = ConvertLanguage $ctx
		#带有行号的翻译
		$commentsnew | Add-Content  $NewFolder$fileNewname"."$lanx".transedcomments.txt"
	}
}


$languages | ForEach-Object {
	$lanx = $_
	$info3 = New-Object System.Collections.ArrayList
	$y = 0
	$commentsnew = Get-Content $NewFolder$fileNewname"."$lanx".transedcomments.txt"
	for ($i = 0; $i -lt $orginfile.count + 1; $i++) {
		if ($commentsnew[$y] -ne $null) {
			$commentscount = $commentsnew[$y].split(',')[0]
		}
		else { $commentscount = "" }
		
		if ($i -eq $commentscount) {
			$info3.Add($commentsnew[$y].split(',')[1]) | Out-Null
			$y++
		}
		else {
			$info3.Add($orginfile[$i]) | Out-Null
		}
	}
	$info3 | Set-Content $NewFolder$fileNewname"."$lanx".ps1" -Force
}


```


```powershell
0,##处理SMB网络的路由，检查网络配置，每个网络的特征
5,#region prep
6,#$ErrorActionPreference = "Stop"
7,#####手动属性
13,#####自动属性
21,#$csvPath = "$rootpath\hfpd01new.csv"
22,#endregion prep
24,#region x
25,#Get-Help about_ActiveDirectory_Filter
26,#刷新组策略
34,# VMM添加运行方式账户
44,# 创建端口分类
46,# 根据端口分类，创建端口配置文件
50,# 创建VM主机组
53,# 创建管理和存储逻辑网络，存储为SMB网络
57,#创建地址池

0,#SMB ネットワークのルーティングの処理、ネットワーク構成の確認、各ネットワークの特性
5,#region prep
6,#$ErrorActionPreference = "Stop"
7,#手動プロパティ
13,#自動プロパティ
21,#$csvPath = "$rootpath\hfpd01new.csv"
22,#endregion prep
24,#region x
25,#ActiveDirectory_Filter に関するヘルプの取得
26,#グループ ポリシーの更新
34,# VMM 実行アカウントの追加
44,# ポート分類の作成
46,# ポートの分類に基づいてポート プロファイルを作成する
50,# VM ホスト グループの作成
53,# SMB ネットワークとして格納される管理およびストレージ論理ネットワークの作成
57,#アドレス プールの作成

```

# 未解决的问题
因为大部分场景已经满足了我自己的需求，比如判断注释里面有没有$,=这些本身有特定含义的行不去翻译。所以还剩下有问题的，都是一些 ~~`很简单`~~ 难搞的部分，比如
1. 无法处理<# #>包裹的注释
2. 无法处理一行代码写完之后不换行，在最后写#然后注释。

如果后面有空，我会继续改进的

用翻译软件是大势所趋，无论是自用还是要走国际范，让代码走出国门，[Microsoft 文本翻译 API](https://azure.microsoft.com/zh-cn/pricing/details/cognitive-services/translator-text-api/)都是您的明智之选
