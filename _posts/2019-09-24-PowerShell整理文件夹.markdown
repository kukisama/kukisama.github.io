# 背景
最近被baidu盘提示说空间不足，想找一找哪些数据是不要的删除一下。


但是baidu盘并没有提供特别快特别好用的全文搜索。所以我的想法是类似CMD一样，把baidu盘上的信息离线下来，然后慢慢分析
```powershell
PS C:\> tree                                     
C:.
├───Intel
│   ├───Logs
│   └───Wireless
│       └───Data
├───PerfLogs
├───Program Files
│   ├───7-Zip
│   │   └───Lang
│   ├───ACD Systems
│   │   └───ACDSee Ultimate
│   │       └───10.0
│   │           ├───1033
│   │           ├───ACDSeeSR Themes
│   │           │   ├───Black
│   │           │   ├───blackshadow
│   │           │   ├───green
│   │           │   ├───grey
│   │           │   ├───transparent
│   │           │   ├───whiteshadow
│   │           │   └───wood
│   │           ├───AlbumGenerator
│   │           │   └───Styles
│   │           │       ├───Style1
│   │           │       │   ├───css

```
# BaiduPCS
BaiduPCS是用GO语言写的，作者大哥的项目在这里 
[戳这里](https://github.com/iikira/BaiduPCS-Go)
这是一个仿linux的cli工具，有了它，列目录这个事情才算有了最基础的平台。
# 需求
BaiduPCS也有tree命令，但是输出效果也是和cmd的tree一样，我需要的实际效果大概和下面类似
```powershell
 !3DS电影/ 3DS 电 影/ 3D 电 影/ 纪 录 片/ 日本风景名胜_400x240_1400kbps_30.303fps_2pass.moflex
 !3DS电影/ 3DS 电 影/ 3D 电 影/ 纪 录 片/ 宇宙的奥秘_400x240_1400kbps_23.976fps_2pass.moflex
 !3DS电影/ 3DS 电 影/ 3D 电 影/ 纪 录 片/ 拯救地球_400x240_1400kbps_29.964fps_2pass.moflex
 !3DS电影/ 3DS 电 影/ 3D 电 影/ 试 机 碟/ 月 光 1 & 2/ 月 光 2_400x240_1400kbps_23.976fps_2pass.moflex
 !3DS电影/ 3DS 电 影/ 3D 电 影/ 试 机 碟/ 月 光 1 & 2/ 月 光_400x240_1400kbps_23.976fps_2pass.moflex
 !3DS电影/ 3DS 电 影/ 3D 电 影/ 试 机 碟/ 月 光 3 & 4/ 月 光 3_400x240_1400kbps_23.976fps_2pass.moflex
 !3DS电影/ 3DS 电 影/ 3D 电 影/ 试 机 碟/ 月 光 3 & 4/ 月 光 4_400x240_1400kbps_24.000fps_2pass.moflex
```
这样找到文件之后，我可以按照目录去baidu盘上删东西。
# 实现
实现代码如下
```powershell
#针对这个目标，主要是处理3种字符
#├出现这个符号，则表示这一行有一个文件名，
#如果这一行末位是/，则它是一个目录。否则，就是一个文件名
#└出现这个符号，表示下一行，会换目录，比如当前是在一个3级目录，下次可能就是2级或者1级
#│出现这个符号，表示这一行的数据处于某一个子目录下，出现一次就是1级目录
#由于要将逻辑放在函数中，所以这里用到了全局变量。
```

```powershell
#为了输出更快，这里将数据放在ArrayList中，最后一次性输出。
$aa = Get-Content C:\333.txt
$aa = $aa.replace("──", "")#把正文中的横杠全删除掉，但是如果文件名中正好有这个字符，就同样会被处理掉
$arrayx = New-Object System.Collections.ArrayList
$global:pathname = ""
```
# 函数主逻辑
下面是关键部分，因为逻辑比较繁琐，写了注释。
```powershell

function sw ($x) {
    $x1 = $x[0] #检测每一行的第一个字符
    #根目录逻辑，如果以├开头，且以/结尾，则表示它是一个根目录。这里全局变量就设置成├以后的部分
    #。例如├── !3DS电影/
    if ($x1 -eq '├' -and $x.EndsWith('/')) {
        $global:pathname = $x.Split('├')[-1]
    }

    #子目录逻辑，如果包含│，且以/结尾，则表示它是一个子目录。这里全局变量需要累加
    #累加原值+“├”以后的部分
    #例如│   │   ├── 3D 电 影/
    if ($x1 -eq '│' -and $x.EndsWith('/') ) {
        $global:pathname = $pathname + $x.Split('├')[-1]
    }
    #子目录中的文件逻辑，如果包含│，且不以/结尾，则表示它是一个子目录的文件
    if ($x1 -eq '│' -and ($x.EndsWith('/') -eq $false)) {
        #首先生成文件名，文件名的格式为全局变量的路径名称+“├或└”右侧的信息
        $global:fname = $global:pathname + $x.Split('├')[-1].Split('└')[-1]
        #然后把数据吐到arraylist中，留到最后统一输出
        $arrayx.add($global:fname) | out-null
        if ($x -match "└") {
            #如果路径中包含└，则需要进行处理，方便下一个待处理数据进行处理。例如如下两条是生成的记录
            #
            #测试数据：
            #│   │   │   │   └── 无 敌 破 坏 王.2012_400x240_2000kbps_23.976fps_2pass.moflex
            #│   │   │   ├── 纪 录 片/
            #│   │   │   │   ├── 风光/
            #│   │   │   │   │   ├── 北碧_400x240_1400kbps_25.000fps_2pass.moflex
            #
            #以下是生成的数据：
            #!3DS电影/ 3DS 电 影/ 3D 电 影/ 电 影 v4/ 无 敌 破 坏 王.2012_400x240_2000kbps_23.976fps_2pass.moflex
            #!3DS电影/ 3DS 电 影/ 3D 电 影/ 纪 录 片/ 风光/ 北碧_400x240_1400kbps_25.000fps_2pass.moflex
            #
            #全局变量以/进行拆解，取0到 $x(未处理数据以│隔开)-3的位置，然后用/拼接回去，再加上/
            #以上面举例，无敌破坏王是$x,当前全局变量是“!3DS电影/ 3DS 电 影/ 3D 电 影/ 电 影 v4/”
            #$x可以被拆分为5个部分，他在一个4级目录下
            #全局变量以/拆解，可以被拆解为5个部分
            #所以全局变量被拆解重新组合后，变成了0..(5-3),也就是到了上一级目录。
            #而这种计算是自动的，所以可以做到从4级目录返回到3级，也可以从3级返回到1级。这里-3是上一级，-2是同级
            #至于返回之后，下一条数据如果是目录，则依然可以返回
            $global:pathname = ($global:pathname.Split('/')[0..(($x.split('│').count - 3) )] -join '/') + '/'
        }
        else {
            $global:pathname = ($global:pathname.Split('/')[0..($x.split('│').count - 2)] -join '/') + '/'
        }
    }
}

$aa | ForEach-Object { sw $_ }
$arrayx | Out-File c:\444.txt
```
# 问题
这里还有一个问题，就是如果是根目录，但是又是文件，应该如何处理。
