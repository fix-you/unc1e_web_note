

# vulhub漏洞复现

[toc]


```shell
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

   

   

## rsync未授权访问 873端口

```[cmd]
rsync rsync://210.14.136.172:873/  /localpath -av
rsync [ip]::
rsync rsync://210.14.136.172:873
```

ref:	https://my.oschina.net/ccLlinux/blog/1859116

**影响范围**

mysql CVE-2012-2122  msf-auxiliary/scanner/mysql/mysql_authbypass_hashdump
All MariaDB and MySQL versions up to 
5.1.61, 
5.2.11, 
5.3.5, 
*5.5.22* 
5.5.23	are	vulnerable.

*MySQL versions from 5.1.63, 5.5.24, 5.6.6 are not.*

*MariaDB versions from 5.1.62, 5.2.12, 5.3.6, 5.5.23 are not.*

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

