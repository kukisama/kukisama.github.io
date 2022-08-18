
### 在Windows中，测试一个目标地址的端口是否通断，常用办法是使用telnet。
### 但是这工具的问题在于不随系统安装，且对端目标端口失败的时候，超时时间很长

```powershell
#找到一个大佬分享的方法

$address = "192.168.1.10"

$port = 80

$tcp = new-object Net.Sockets.TcpClient

$tcp.Connect($address,$port)

If ($? -ne "True")

   {Write-Host $address"的端口"$port"连接失败"}

Else {$tcp.Close()}
#原作者
#https://blog.51cto.com/beanxyz/1784596

```
> 这里有个问题就是，如果连接失败，超时时间特别长，而且会有红色报错，根据关键字Net.Sockets.TcpClient继续查找资料

> 去官方网站看看这个方法
https://docs.microsoft.com/en-us/dotnet/api/system.net.sockets.tcpclient?view=netframework-4.8

> 这里介绍的很详细，不过主要问题是可以使用的方法太多，不知道用哪一个。

在Stackoverflow找到最合适的一个例子
[这里](https://stackoverflow.com/questions/11837541/check-if-a-port-is-open)

```csharp
#原C#逻辑

bool IsPortOpen(string host, int port, TimeSpan timeout)
{
    try
    {
        using(var client = new TcpClient())
        {
            var result = client.BeginConnect(host, port, null, null);
            var success = result.AsyncWaitHandle.WaitOne(timeout);
###下文省略
```

### 翻译一下，变成我们PowerShell的写法，问题解决，还可以设置超时时间，速度非常快
```powershell
$tcp = new-object Net.Sockets.TcpClient
$result = $tcp.BeginConnect("baidu.com", "80", $null,$null)
$result.AsyncWaitHandle.WaitOne(200)
$tcp.Close()
```
