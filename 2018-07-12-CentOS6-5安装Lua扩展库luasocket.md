---
title: CentOS6.5安装Lua扩展库luasocket
date: 2018-07-12 15:02:52
categories: "编程"
tags: [Lua]
---

# 下载安装包
LuaJIT安装包：http://luajit.org/download/LuaJIT-2.0.4.tar.gz
luasocket扩展库：http://files.luaforge.net/releases/luasocket/luasocket/luasocket-2.0.2/luasocket-2.0.2.tar.gz

# 安装过程
## 安装LuaJIT

	tar -zxf LuaJIT-2.0.4.tar.gz 
	cd LuaJIT-2.0.4/
	make && make install
	echo '/usr/local/lib’ >>/etc/ld.so.conf
	ldconfig

<!-- more -->
## 安装luasocket
### 解压安装包

	tar -zxf luasocket-2.0.2.tar.gz
	cd luasocket-2.0.2/
	
### 修改配置文件如下并安装
	cat config |grep -viP "^#|^$"
	EXT=so
	SOCKET_V=2.0.2
	MIME_V=1.0.2
	SOCKET_SO=socket.$(EXT).$(SOCKET_V) 
	MIME_SO=mime.$(EXT).$(MIME_V)
	UNIX_SO=unix.$(EXT)
	LUAINC=-I/usr/local/include/luajit-2.0
	INSTALL_TOP_SHARE=/usr/local/share/lua/5.1
	INSTALL_TOP_LIB=/usr/local/lib/lua/5.1
	INSTALL_DATA=cp
	INSTALL_EXEC=cp
	CC=gcc
	DEF=-DLUASOCKET_DEBUG 
	CFLAGS= $(LUAINC) $(DEF) -pedantic -Wall -O2 -fpic
	LDFLAGS=-O -shared -fpic
	LD=gcc 
	[root@huidu-inrouter-nginx-079007 luasocket-2.0.2]# 
 	make && make install 
 	
# 安装成功验证
	]# lua
	Lua 5.1.4  Copyright (C) 1994-2008 Lua.org, PUC-Rio
	> require("socket")
	>

# 遇到问题
require("socket")时会报错，把需要的socket.lua 和core.so 拷到对应的目录即可使用；

	cp /usr/local/share/lua/5.1/socket.lua /usr/share/lua/5.1/
	cp /usr/local/lib/lua/5.1/socket/core.so /usr/lib64/lua/5.1/socket/
