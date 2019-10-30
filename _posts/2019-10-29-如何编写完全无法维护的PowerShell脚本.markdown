###### 说起无法维护的代码，相信朋友们有很多的想法，比如说没有注释、屎一样的逻辑、通篇重复代码

但是下面这个方法一定是最彻底最让你绝望的。

---
## 方法
朋友们一定会对变量的命名下很多心思，如果不认真对待变量名，后果是致命的。
下面是把一个现有的ps1文件，简单的转换成作死代码。

```powershell
$gcfile = Get-Content "C:\x\azaa.ps1"
$xyz = ($gcfile -match "^\$.*=?" | ForEach-Object { $_.split(' |%|+|=')[0] } | Select-Object -Unique )
$xyz | Sort-Object -Property length -Descending
$字母小写 = 0..25 | ForEach-Object { [char][int](0x0061 + (0x01 * $_)) }
$日文 = 0..50 | ForEach-Object { [char][int](0x306e + (0x01 * $_)) }
$字库=$日文
0..$($xyz.count - 1) | ForEach-Object {
    $改造变量名 = "$"
    1..8 | ForEach-Object { $改造变量名 += Get-Random $字库 }
    $gcfile = $gcfile.replace($xyz[$_], $改造变量名)
}
$gcfile
```
我们来看看原文

```powershell
$postParams = @{"Ocp-Apim-Subscription-Key" = "$subscriptionKey" }
$comments = Get-Content $NewFolder$fileNewname".comments.txt"   -Encoding utf8
$body = $comments | % { $_ | Select-Object @{name = 'text'; e = { $_.split(',')[-1].replace('#', '') } } } | ConvertTo-Json

$languages|%{
$lanx=$_

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
加工后的代码
```powershell
$みぶ゗をわやべ゙ = @{"Ocp-Apim-Subscription-Key" = "$るゐを゙ぱゕゕの" }
$んゃゅるぽらゃほ = Get-Content $りぽわめる゗れ゗$めゆびわゃふみぱ".comments.txt"   -Encoding utf8
$ろゑらぴぶゎやひ = $んゃゅるぽらゃほ | % { $_ | Select-Object @{name = 'text'; e = { $_.split(',')[-1].replace('#', '') } } } | ConvertTo-Json

$まみゎわゕゞぷゎ|%{
$むゐへやゅゎるゟ=$_
$ぽ゚ぷ゜ぱも゠ぽ = translatorMS -language $むゐへやゅゎるゟ -texts $ろゑらぴぶゎやひ
$んゃゅるぽらゃほnew = ConvertLanguage
    $ひへぱび゗゙のゆ3.Add($んゃゅるぽらゃほnew[$ゐりへゟゃ゛ゎを].split(',')[1]) | Out-Null
    $ゐりへゟゃ゛ゎを++
  }
  else {
    $ひへぱび゗゙のゆ3.Add($びほゎばべゕび゙[$i]) | Out-Null
  }
}
$ひへぱび゗゙のゆ3 | Set-Content $りぽわめる゗れ゗$めゆびわゃふみぱ"."$むゐへやゅゎるゟ".ps1" -Force
}

```
当然这是随机的，所以每次输出都可以不一样，英韩俄印总有一款适合你

```powershell
$ゃ゙ぼぷれゃぶも = @{"Ocp-Apim-Subscription-Key" = "$ょべゆら゠べゃみ" }
$ぽめ゚ゐをれまゔ = Get-Content $ゅぺんぴばひひゃ$ゃや゘をんびはぺ".comments.txt"   -Encoding utf8
$ぶぶわやゔゑめ゜ = $ぽめ゚ゐをれまゔ | % { $_ | Select-Object @{name = 'text'; e = { $_.split(',')[-1].replace('#', '') } } } | Conver
tTo-Json

$ゆゐばみゃゐのの|%{
$ばゔをゔゝょゎら=$_
$ゖ゗ふぺゃゑまり = translatorMS -language $ばゔをゔゝょゎら -texts $ぶぶわやゔゑめ゜
  else {
    $やゝもわぱ゜ゃば3.Add($べみ゠ぷまゖょび[$i]) | Out-Null
  }
}
$やゝもわぱ゜ゃば3 | Set-Content $ゅぺんぴばひひゃ$ゃや゘をんびはぺ"."$ばゔをゔゝょゎら".ps1" -Force
}
```

## 老实说，作为这段代码的亲身父亲，我已经放弃跟他相认。
## 如果有下一回合的话，我们会介绍一下为什么要这么做，以及这篇文章中用到的一些有用技术点。
