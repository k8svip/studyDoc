---
title: nginx的location配置验证
date: 2018-07-20 13:28:05
categories: "中间件"
tags: [nginx]
---

# 简介
个人从事运维使用nginx有一段时间，不敢说精通，但常规问题可以解决，在工作中有很多同学对locaton的匹配规则，不是很了解，经常跑过来问怎么这个规则没有跑，哪个规则出问题了，他们也希望做一个针对性培训，其实网络上面的文章已经很多很多，我这里也参考网络上的👀，有错的地方可以指出来

# 匹配语法
nginx的url匹配模式很强大，使用非常灵活，这里如果不寻找规律，有些地方很懵圈，下面就是nginx的location 相关的所有语法；

<!-- more -->

```shell
location [ = | ~ | ~* | ^~ ] uri { ... }
location @name { ... }
```
是的，语法就这一点，是不是感觉很简单呢？

```shell
location [指令模式] uri {
...
}
```

# 匹配模式
这里的指令模式有[ = | ~ | ~* | ^~ ]， 根据不同的指令模式，我们分为：
精确匹配： = uri { ... }
前缀匹配： ^~ uri { ... }
正则匹配： ~ uri { ... } 和 ~* uri { ... }
正常匹配：  uri { ... }
全匹配：	/ { ... } 

提前说结论： 精确匹配 > 前缀匹配 > 正则匹配 > 正常匹配 > 全匹配

#匹配详解
## 精确匹配

```shell
## 第1段
location  /jdb/ {
	root /data/nginx/html4/;
	index index.html;
}

## 第2段
location = /jdb/ {
	root /data/nginx/html3/;
	index index.html;
}
```
访问：http://127.0.0.1/jdb/


根据这个匹配结论，这里应该显示什么呢？大家都会认为是第二段代码中的 = /jdb/ 你可以实验下，不是的，是第一段，为什么？

- 精确匹配中 '/jdb/'中优先匹配到第二段；
- 再访问'/jdb/index.html'，此次内部跳转uri已经是'/jdb/index.html'而非 =的；
- 最终访问结果是第1段中的index.html;

下面我们再看一个例子

```shell
## 第3段
location  /jdb/ {
	rewrite ^/jdb/$ https://www.jiedaibao.com/ break;
}

## 第4段
location = /jdb/ {
	rewrite ^/jdb/$ https://www.sina.com.cn/ break;
}
```

访问：http://127.0.0.1/jdb/ 看下效果，是不是匹配到第4段了呢,答案：是的；
访问http://127.0.0.1/jdB/ ，http://127.0.0.1/jdb/abc看下效果，答案：都无法正常访问

结论：精确匹配区分大小写，不能使用正则，访问的URI必须完全与=后面的一致，多一个"/"或者少一个"/"，都是不可以的；

## 前缀匹配
```shell
location ^~ /jdb/ {
	rewrite ^  https://www.163.com break;
}

location ^~ /jdb/bcd/ {
	rewrite ^  https://www.qq.com break;
}

location ^~ /Abc/ {
	rewrite ^  https://www.sina.com.cn break;
}
```

- 访问：http://127.0.0.1/jdb/ 成功，
- 访问：http://127.0.0.1/jdb/abcd/成功，只匹配前缀；
- 访问：http://127.0.0.1/Jdb/ 不成功；不支持大小写；
- 访问：http://127.0.0.1/jdB/123 不成功；
- 访问：http://127.0.0.1/Abc/ 成功；
- 访问：http://127.0.0.1/Abc/abc 成功；

结论：前缀匹配不能使用正则，区分大小写，只要前缀相同，都可以匹配成功，不管后面有没有字符，保证前缀相同即可；
## 正则匹配
```shell
location ~ /[a-z]jdb/ {
	rewrite ^  https://www.sina.com.cn break;
}

location ~* /[a-z]jdb/ {
	rewrite ^  https://www.google.com break;
}
```

- 访问：http://127.0.0.1/ajdb/ 成功，正则匹配；
- 访问：http://127.0.0.1/ajdb/bbb 成功，正则+前缀匹配；
- 访问：http://127.0.0.1/zjdb/ 成功；
- 访问：http://127.0.0.1/Ajdb/aaa 成功；

结论：~ 区分大小写，~* 不区分大小写,  并且与前缀匹配比较类似，只需要匹配模式开头部分，这两种同时存在时，优先匹配区分大小写的；

## 正常匹配
```shell
## 第1段
location /jdb/ {
	rewrite ^  https://www.google.com break;
}

## 第2段
location /[0-9]jdb/ {
	rewrite ^  https://www.google.com break;
}
```

- 访问：http://127.0.0.1/jdb/ 成功
- 访问：http://127.0.0.1/jdb/2 成功，类似前缀匹配；
- 访问：http://127.0.0.1/jDB/ 不成功，大小写不支持；
- 访问：http://127.0.0.1/0jdb/ 不成功，正则不支持；


指令模式为空的匹配规则叫正常匹配；


结论：有些文档中说正常匹配支持正则，不区分大小写，个人测试了下，不支持正则，但在uri后面继续跟字符，区分大小写（相信实验结果😀，大家可以测试，如果有问题，可以交流）正常匹配与前辍匹配的差别，只在于优先级；

```shell
## 第3段
location ^~ /jdb/ {
	rewrite ^/jdb/$ https://www.baidu.com/ break;
}

## 第4段
location /jdb/ {
	rewrite ^/jdb/$ https://www.sina.com.cn/ break;
}
```
说明：第3段前缀匹配与第4段正常匹配不能同时存在，否则会报nginx: [emerg] duplicate location "/jdb/" in /data/nginx//conf/nginx.conf:47 因为这两个都是普通字符串；

## 全匹配
```shell
location / {
	root /data/nginx/html;
	index  index.html index.htm;
}
```
说明： 没有匹配指令，并且匹配的URI仅一个斜杠 /, 通常用在一个默认页面的地方，这个地方没有什么好说的；

## 命名匹配
```shell
error_page  404 = @notfound;
location @notfound {
	rewrite ^  https://www.google.com break;
}
```
说明：一般用于静态页面或者错误页面，并且这个命名匹配中，不允许有alias；
# 综合验证优先级实验
```shell
## 全匹配，这里/data/nginx/html/下面有一个jdb文件夹，里面有index.html
location / {
	root /data/nginx/html;
	index index.html;
}

## 正常匹配
location /jdb/ {
	rewrite ^/jdb/$ https://www.sina.com.cn/ break;
}

## 正则匹配
location ~ /[a-z]db/ {
	rewrite ^/jdb/$ https://www.google.com/ break;
}

## 前缀匹配
location ^~ /jdb/ {
	rewrite ^/jdb/$ https://www.baidu.com/ break;
}

## 精确匹配
location = /jdb/ {
	rewrite ^/jdb/$ https://www.jiedaibao.com/ break;
}
```

优先级： 精确匹配 > 前缀匹配 > 正则匹配 > 正常匹配 > 全匹配
 
测试方法：把最先匹配的注释掉，然后再继续测试，就证明了上面的优先级
，注意前缀匹配与正常匹配在相同URI时，不能同时存在；


说明：为了证明以上的优先级顺序, 我们设置了以上的用例，为了更进一步说明问题;


我们把优先级最高的放在配置文件的最下面，这样避免从上到下优先匹配的问题；


匹配原则除了这个优先级外，还有一个就是在相同指令模式匹配中，匹配度最大的URI优先；

	
## 测试说明
测试时，最好不要使用浏览器，因为浏览器有缓存，如果要使用请先清除缓存，或者使用curl或者Postman工具等；
