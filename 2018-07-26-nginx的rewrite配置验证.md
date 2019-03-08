---
title: nginx的rewrite配置验证
date: 2018-07-26 15:24:20
categories: "中间件"
tags: [nginx]
---

# HTTP请求流程

## nginx http请求的11个阶段

有人说，我们不是讲rewrite吗，为什么讲到http请求处理的11个阶段，如果想深入的了解rewrite，就必须对http请求处理的各个阶段有一定的了解，并且如果读nginx源码，也会更加的容易；

## 11个Phase如图

!["nginx处理http请求的11个阶段"](/img/nginx11phase.png)

红色是与REWRITE相关的几个阶段

<!-- more -->
# rewrite配置说明

## 语法

```shell
Syntax:	rewrite regex replacement [flag];
Default:	—
Context:	server, location, if
```

### rewrite官方解释

If the specified regular expression matches a request URI, URI is changed as specified in the replacement string. The rewrite directives are executed sequentially in order of their appearance in the configuration file. It is possible to terminate further processing of the directives using flags. If a replacement string starts with "http://", "https://", or "$scheme", the processing stops and the redirect is returned to a client.

如果指定的正则表达式匹配一个请求的URI，则URI就按照指定的replacement进行重写或重定向，而rewrite指令会按照配置文件中出现的顺序执行，并且flag标志位可以停止处理；如果replacement以"http://"或"https://"或"$scheme"开头，此时停止处理并直接重定向返回客户端；

### rewrite语法解释

regex: 对请求的URI做正则匹配，regex匹配的是URI，不包括域名socket和查询参数，默认的查询参数会被追加到replacement末尾，如果不希望在末尾追加query string，可以在replacement的末尾加一个"?"；

replacement: 目标URI匹配成功后替换的URL；

flag: 控制指令的执行顺序,如果没有flag，rewrite会根据指令出现的先后顺序；

无flag: 无flag时，rewrite匹配顺序由上到下依次匹配；

rewrite_log指令：,日志文件中会记录rewrite匹配的过程，注意日志是记录在error.log中，可以调试的时候打开debug级别查看匹配过程；

### flag有哪些？

#### last
stops processing the current set of ngx_http_rewrite_module directives and starts a search for a new location matching the changed URI;

如果匹配的URI，rewrite在server块中，并且last做为flag，匹配到此rewrite URI时，不再向下匹配server块中的rewrite，进而继续下面location URI的查找匹配；

如果匹配的URI，rewrite在location块中，last做为flag，匹配到此rewrite时，会跳出此location块，继续从上到下查找其它的location块URI，但不会匹配server块中的rewrite中的URI；

#### break

stops processing the current set of ngx_http_rewrite_module directives as with the break directive;

如果匹配的URI，rewrite在server块中，并且last做为break，匹配到此rewrite URI时，不再向下匹配server块中的rewrite，进而继续下面location URI的查找匹配；

如果匹配的URI，rewrite在location块中，last做为break，匹配到此rewrite时，不会跳出此location块，而是继续对location块下面的语句继续运行，不会跳出此location块，并且也不会匹配location 块下面的rewrite；

完成该rewrite规则的执行后，停止处理后续rewrite指令集，并不再重新查找；但是当前location内剩余非rewrite语句和location外的的非rewrite语句可以执行；

#### redirect
returns a temporary redirect with the 302 code; used if a replacement string does not start with "http://","https://", 'scheme';(这里的scheme前面有一个￥)

返回302临时重定向，地址栏会显示跳转后的地址；

#### permanent

returns a permanent redirect with the 301 code.

返回301永久重定向，地址栏会显示跳转后的地址，即表示如果客户端不清理浏览器缓存，那么返回的结果将永久保存在客户端浏览器中了。

# rewrite配置实例

## 无flag_案例1

```shell
server {
    listen       80;

    rewrite ^/(.*)$         /jdb_one/$1;
    rewrite ^/jdb_one/(.*)$  /jdb_two/$1;

    location / {
        echo "This is default location";
        echo "uri: ${uri}";
    }

    location /jdb_one/ {
        echo "This is jdb_one location";
        echo "uri: ${uri}";
    }

    location /jdb_two/ {
        echo "This is jdb_two location";
        echo "uri: ${uri}";
    }
}
```

测试：curl http://127.0.0.1/abc 

回显：

```shell
This is jdb_two location
uri: /jdb_two/abc
```

说明：

1. 无flag，会依赖向下匹配，根据nginx在http请求处理的阶段中，我们会先匹配server块中的rewrite规则；
2. 第一次匹配rewrite ^/(.*)$         /jdb\_one/$1; URI变成:/jdb\_one/abc，无flag继续向下匹配；
3. 第二次匹配rewrite ^/jdb\_one/(.*)$  /jdb\_two/$1; URI变成/jdb\_two/abc；
4. 再向下FIND\_CONFIG阶段，查找location进行匹配，正好找到location /jdb\_two/ 所以回显如上；

## 无flag_案例2
```shell
server {
    listen       80;

    rewrite ^/(.*)$         /jdb_one/$1;
    location / {
        echo "This is default location";
        echo "uri: ${uri}";
    }
    location /jdb_one/ {
        rewrite ^/jdb_one/(.*)$  /jdb_two/$1;
        echo "This is jdb_one location";
        echo "uri: ${uri}";
    }
    location /jdb_two/ {
        echo "This is jdb_two location";
        echo "uri: ${uri}";
    }
}
```

测试：curl http://127.0.0.1/abc
回显：

```shell
This is jdb_two location
uri: /jdb_two/abc
```
说明：

1. 无flag，处理server块阶段，匹配rewrite ^/(.*)$         /jdb\_one/$1 ，URI为：/jdb\_one/abc;
2. 到FIND\_CONFIG阶段，匹配location, location /jdb\_one/ ,这个location块中有rewrite再次匹配^/jdb\_one/(.*)$  /jdb\_two/$1; URI变为：/jdb\_two/abc，这里没有flag，会跳出继续FIND\_CONFIG阶段，而不会到SERVER\_REWRITE阶段；
3. 匹配location /jdb\_two/ ，所以回显如上;

## flag redirect 案例1

```shell
server {
    listen       80;

    #rewrite ^/(.*)$         /jdb_one/$1 redirect;
    rewrite ^/(.*)$         https://www.baidu.com/ redirect;
    rewrite ^/jdb_one/(.*)$  /jdb_two/$1;
    rewrite ^/jdb_two/(.*)$  /jdb_three/$1;

    location / {
        echo "This is default location";
		  echo "uri: ${uri}";
    }

    location /jdb_one/ {
        echo "This is jdb_one location";
		 echo "uri: ${uri}";
    }

    location /jdb_two/ {
        echo "This is jdb_two location";
		 echo "uri: ${uri}";
    }

    location /jdb_three/ {
        echo "This is jdb_three location";
		 echo "uri: ${uri}";
    }
}
```

测试：浏览器访问 http://127.0.0.1/abc

回显：跳转到百度，状态码为302，临时重定向；

说明：

1. 这里的flag是redirect，说明需要重定向到replacement, 正好这里的replacement有“https://”，此时会直接跳转并返回给客户端；
2.  如果这里的#rewrite ^/(.*)$         /jdb\_one/$1 redirect; 这里的注释#打开，会出现问题情况呢？使用浏览器测试下，并自行分析下，死循环；如果不明白，可以联系我；

## flag redirect 案例2

```shell
server {
    listen       80;
    server_name a.com;
    rewrite_log on;
    rewrite ^/jdb_one/(.*)$  /jdb_two/$1;
    rewrite ^/jdb_two/(.*)$  /jdb_three/$1 redirect;
    rewrite ^/(.*)$         /jdb_one/$1;

    location / {
        echo "This is default location";
        echo "uri: ${uri}";
    }

    location /jdb_one/ {
         echo "This is jdb_one location";
		 echo "uri: ${uri}";
    }

    location /jdb_two/ {
        echo "This is jdb_two location";
		echo "uri: ${uri}";
    }

    location /jdb_three/ {
        echo "This is jdb_three location";
		echo "uri: ${uri}";
    }
}
```
测试：浏览器访问 http://127.0.0.1/jdb_one/abc 

浏览器：跳转为：http://127.0.0.1/jdb_three/abc

浏览器回显：

```shell
This is jdb_one location
uri: /jdb_one/jdb_three/abc
```
说明：

1. 匹配rewrite ^/jdb\_one/(.*)$  /jdb\_two/$1; 后，URI变为：/jdb\_two/abc ; 无flag，继续向下；
2. 继续匹配 rewrite ^/jdb\_two/(.*)$  /jdb\_three/$1 redirect; 302临时重定向，URI: http://127.0.0.1/jdb\_three/abc, 
3. 再次访问，此时会从SERVER\_REWRITE这个阶段开始，此时匹配的是 rewrite ^/(.*)$         /jdb\_one/$1; URI变为：/jdb\_one/jdb\_three/abc ，无flag，向下FIND\_CONFIG阶段；
4. 查看location 匹配location /jdb\_one/ ,所以回显如上；


## flag permanent案例

1. 其实这里的案例与redirect是一模一样的，只是状态码不一样，redirect 302 临时重定向, permanent 301 永久重定向，并且两种重定向，浏览器URL都会发现跳转，;
2. 其实301和302重定向时，也有很多人称为外部重定向，这里的意思是他会重新从nginx http请求处理的第一个阶段开始重新匹配，而相对的内部重定向，是指不会到SERVER\_REWRITE阶段的重定向，但有可能是REWRITE和FIND\_CONFIG这两个阶段的相互跳转查找，在flag break和last的时候会讲解到；

## flag last案例1
```shell
server {
    listen       80;
    rewrite_log on;
    rewrite ^/jdb_one/(.*)$  /jdb_two/$1 last;
    rewrite ^/jdb_two/(.*)$  /jdb_three/$1;

    location / {
        echo "This is default location";
		echo "uri: ${uri}";
    }

    location /jdb_one/ {
        echo "This is jdb_one location";
		echo "uri: ${uri}";
    }

    location /jdb_two/ {
        echo "This is jdb_two location";
		echo "uri: ${uri}";
    }

    location /jdb_three/ {
        echo "This is jdb_three location";
		echo "uri: ${uri}";
    }
}
```

测试： curl http://127.0.0.1/jdb\_one/abc

回显：

```shell
This is jdb_two location
uri: /jdb_two/abc
```
说明：

1. 匹配 rewrite ^/jdb\_one/(.*)$  /jdb\_two/$1 last;遇到last，停止同级段的匹配，这里的意思是，中止server段向下的匹配，进行FIND\_CONFIG阶段，URI变为：/jdb\_two/abc; 
2. 这里可以看出，如果last没有中止server段向下的匹配，会匹配rewrite ^/jdb\_two/(.*)$  /jdb\_three/$1; 实际结果是没有匹配的；
3. 由1之后，匹配location /jdb\_two/ 所以会出现上面的回显结果;


## flag last案例2
```shell
server {
    listen       80;
    server_name a.com;
    rewrite_log on;

    rewrite ^/(.*)$  /jdb_one/$1;
    rewrite ^/jdb_three/(.*)$ /jdb_four/$1;

    location / {
        echo "This is default location";
		echo "uri: ${uri}";
    }

    location /jdb_one/ {
        rewrite ^/(.*)$             /jdb_three/$1 last;
        rewrite ^/jdb_three/(.*)$    /;

        echo "This is jdb_one location";
		echo "uri: ${uri}";
    }

    location /jdb_two/ {
        echo "This is jdb_two location";
		echo "uri: ${uri}";
    }

    location /jdb_three/ {
        echo "This is jdb_three location";
		echo "uri: ${uri}";
    }

    location /jdb_four/ {
        echo "This is jdb_four location";
        echo "uri: ${uri}";
    }
}
```

测试：curl http://127.0.0.1/abc

回显：

```shell
This is jdb_three location
uri: /jdb_three/jdb_one/abc
```

说明：

1. 匹配rewrite ^/(.*)$  /jdb\_one/$1; URI变为：/jdb\_one/abc;
2. 匹配location /jdb\_one/ 进而匹配location块中的rewrite ^/(.*)$             /jdb\_three/$1 last; 因为是last，会跳出location继续FIND\_CONFIG阶段，URI为：/jdb\_three/jdb\_one/abc；而不会到SERVER\_WRITE阶段；
3. 匹配 location /jdb\_three/ ,所以看到上面的回显；

## flag break案例1
```shell
server {
    listen       80;
    rewrite_log on;
    rewrite ^/jdb_one/(.*)$  /jdb_two/$1 break;
    rewrite ^/jdb_two/(.*)$  /jdb_three/$1;

    location / {
        echo "This is default location";
		echo "uri: ${uri}";
    }

    location /jdb_one/ {
        echo "This is jdb_one location";
		echo "uri: ${uri}";
    }

    location /jdb_two/ {
        echo "This is jdb_two location";
		echo "uri: ${uri}";
    }

    location /jdb_three/ {
        echo "This is jdb_three location";
		echo "uri: ${uri}";
    }
}
```
测试：curl http://127.0.0.1/jdb_one/abc

回显：

```shell
This is jdb_two location
uri: /jdb_two/abc
```
说明：

1. 匹配rewrite ^/jdb_one/(.*)$  /jdb\_two/$1 break; flag为break结束本层级的rewirte匹配，URI变为：/jdb\_two/abc
2. 继续FIND\_CONFIG阶段，匹配location /jdb\_two/ ; 所以回显如上；

## flag break案例2
```shell
server {
    listen       80;
    server_name a.com;
    rewrite_log on;

    rewrite ^/(.*)$  /jdb_one/$1;
    location / {
        echo "This is default location";
        echo "uri: ${uri}";
    }

    location /jdb_one/ {
        rewrite ^/(.*)$         /jdb_three/$1 break;
        rewrite ^/jdb_two/(.*)$  /;

        echo "This is jdb_one location";
        echo "uri: ${uri}";
    }

    location /jdb_two/ {
        echo "This is jdb_two location";
        echo "uri: ${uri}";
    }

    location /jdb_three/ {
        echo "This is jdb_three location";
		echo "uri: ${uri}";
    }
}
```

测试：curl http://127.0.0.1/abc

回显：

```shell
This is jdb_one location
uri: /jdb_three/jdb_one/abc
```
说明：

1. 匹配rewrite ^/(.*)$  /jdb\_one/$1; URI变为：/jdb\_one/abc
2. 匹配location /jdb\_one/，然后继续 rewrite ^/(.*)$         /jdb\_three/$1 break; flag为break，结束本层级的rewrite ^/(.*)$         /jdb\_three/$1 break;，并且继续进行本层级的其它操作；
3. 此时的URI：/jdb\_three/jdb\_one/abc,所以回显如上；

