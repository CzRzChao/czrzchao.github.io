---
title: lnmp安装手册
date: 2017-11-14 16:48:30
categories: 
- lnmp
tags: 
- php
- linux
- mysql
- nginx
---

本文主要是通过针对lnmp的环境搭建做一个系统性的总结，方便用时查找。

# 概述
最流行的web研发生态当属Linux+Nginx+Mysql+PHP了，此四者均开源免费，且社区活跃，是web开发的利器。俗话说，工欲善其事，必先利其器。lnmp架构是业务的基础，虽然在企业中环境都由运维部门接管，业务RD不需要为此操心，但业务运行在什么样的环境下，会遇到什么样的问题，作为业务RD却又不得不去弄清楚他们之间的关系。  

此文主要以实际着手搭建一套lnmp环境为主线，介绍其中涉及到的技术点。包括php配置、php-fpm配置、nginx配置、mysql配置。以及这他们之间的关系。

***

# 准备工作
## 源码下载
1. Linux: 本文是基于CentOS6.5
2. Nginx: 1.13.4，下载地址：[http://nginx.org/download/nginx-1.13.4.tar.gz](http://nginx.org/download/nginx-1.13.4.tar.gz)
3. Mysql: 5.6.37，下载地址：[https://dev.mysql.com/get/Downloads/MySQL-5.6/mysql-5.6.37.tar.gz](https://dev.mysql.com/get/Downloads/MySQL-5.6/mysql-5.6.37.tar.gz)
4. PHP: 5.5.38，下载地址：[http://cn2.php.net/distributions/php-5.5.38.tar.gz](http://cn2.php.net/distributions/php-5.5.38.tar.gz)

在/usr/local目录下创建我们的LNMP安装目录，我们这里定义为lnmp  

```shell
mkdir -p /usr/local/lnmp/src
wget -P /usr/local/lnmp/src http://nginx.org/download/nginx-1.13.4.tar.gz
wget -P /usr/local/lnmp/src https://github.com/mysql/mysql-server/archive/mysql-5.6.37.tar.gz
wget -P /usr/local/lnmp/src http://cn2.php.net/distributions/php-5.5.38.tar.gz
```

为了方便知道我们安装的软件版本，故目录命名均采用软件名-三位版本号的形式：  

```shell
mkdir /usr/local/lnmp/php-5.5.38
mkdir /usr/local/lnmp/mysql-5.6.37
mkdir /usr/local/lnmp/nginx-1.10.3
```

## 工具准备

本问所有的安装都是根据源码进行编译安装，需要使用到的工具有:  

### cmake
cmake是一款开源跨平台的编译工具，其包含编译构建、测试打包等一体的工具包。mysql5.5以上的版本都要通过cmake进行编译安装。

* 官网: [https://cmake.org/](https://cmake.org/)
* 下载地址: [https://cmake.org/files/v3.9/cmake-3.9.1.tar.gz](https://cmake.org/files/v3.9/cmake-3.9.1.tar.gz)
* github: [https://github.com/Kitware/CMake/archive/v3.9.1.tar.gz](https://github.com/Kitware/CMake/archive/v3.9.1.tar.gz)

### GCC(version >= 4.2.1)
作为RD应该都或多或少的了解过GCC（the GNU Compiler Collection），即GNU编译套件集合，是由 GNU 开发的编程语言编译器。它是以GPL许可证所发行的自由软件。其包括C、C++、Objective-C、Fortran、Java、Ada和Go语言的前端，也包括了这些语言的库（如libstdc++、libgcj等等）。这里给出官网地址: [http://gcc.gnu.org/](http://gcc.gnu.org/)

### make(version >= 3.75)
又叫GNU Make，此软件是GNU系列的编译软件。在linux下编译软件几乎是必不可少的。

* 官网: [http://www.gnu.org/software/make/](http://www.gnu.org/software/make/)
* 使用手册: [http://www.gnu.org/software/make/manual/make.html](http://www.gnu.org/software/make/manual/make.html)

> mysql源码编译安装依赖 [https://dev.mysql.com/doc/refman/5.6/en/source-installation.html](https://dev.mysql.com/doc/refman/5.6/en/source-installation.html)

***
# Nginx安装
## 编译安装

```shell
#解压源码文件
cd /usr/local/lnmp/src
tar -zxf nginx-1.13.4.tar.gz
cd nginx-1.13.4
​
#执行configure，指定安装目录为 /usr/local/lnmp/nginx-1.13.4
./configure --prefix=/usr/local/lnmp/nginx-1.13.4
make && make install
​
#进入nginx安装目录，检测是否安装成功
cd /usr/local/lnmp/nginx-1.13.4
​
#执行下列命令，若出现错误，跟进错误进行修正
nginx -t 
```

## 软件配置
本文只介绍最简单的基本配置，主要是为了让环境能够正常跑起来

### nginx.conf

```shell
#user  nobody;
worker_processes  1;
​
pid logs/nginx.pid;
​
events {
    worker_connections  1024;
}
​
http {
    include       mime.types;
    default_type  application/octet-stream;
    #日志格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
​
    sendfile        on;
    keepalive_timeout  65;
​
    #gzip  on;
    include vhost/*.conf;
}
```

### vhost
一套lnmp环境下可能运行不止一个服务，一次我们在nginx的conf目录下创建vhost目录，用于存放虚拟主机。

```shell
server {
    listen       8000;
    server_name  localhost;
    access_log  logs/host.access.log  main;
​
    location / {
        root   /data0/www/htdocs/lnmp.com;
        index  index.php;
    }
​
    #error_page  404              /404.html;
​
    # redirect server error pages to the static page /50x.html
    error_page   500 502 503 504  /50x.html;
​
    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    location ~ \.php$ {
        root           /data0/www/htdocs/lnmp.com;
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }
}
```

## 测试
检测安装是否成功，进入 `/usr/local/lnmp/nginx-1.13.4/sbin`目录，执行`./nginx -t`命令，若输出如下信息，则表示安装成功。

```shell
nginx: the configuration file /usr/local/lnmp/nginx-1.13.4/conf/nginx.conf syntax is ok nginx: configuration file /usr/local/lnmp/nginx-1.13.4/conf/nginx.conf test is successful
```

## 使用
nginx安装完成后，目录如下：

```shell
.
├── client_body_temp
├── conf //此目录为存放配置文件目录
│   └── vhost
├── fastcgi_temp
├── html //存放网站代码的目录，一般情况下，我们会有自己的网站目录，这里可以存放一些统一的4xx、5xx页面。
# ├── logs //默认存放nginx运行日志的地方
├── proxy_temp
├── sbin //此目录是存放执行命令的目录
├── scgi_temp
└── uwsgi_temp
```

### sbin
这里我们先说sbin目录，其中只有一个文件nginx可执行文件，此文件用于管理nginx的启动、停止、重启。

nginx命令支持以下参数：

* -v ：显示版本号并退出
* -V：显示版本号，同时显示编译时选项并退出
* -t：测试配置文件，并退出
* -T：测试配置文件，并将其输出，然后退出
* -s signal：发送信号给nginx master，用于平滑停止，退出，平滑重启，重启。signal包括（stop\|quit\|reload\|reopen）
* -c：指定nginx的配置文件，默认为conf/nginx.conf
* -g：设置配置文件之外的全局指令，用的比较少

#### nginx启动
使用默认的配置文件：

```shell
/usr/local/lnmp/nginx-1.13.4/sbin/nginx 
```

使用指定的配置文件：

```shell
/usr/local/lnmp/nginx-1.13.4/sbin/nginx -c /path/yourConfigFile
```

#### nginx停止

```shell
/usr/local/lnmp/nginx-1.13.4/sbin/nginx -s stop
```
对于nginx停止还可以使用kill命令

```shell
kill -QUIT 主进程pid 或 kill -QUIT `cat /path/pid` # 从容关闭
kill -TERM 主进程pid 或 kill -TERM `cat /path/pid` # 快速关闭
kill -INT 主进程pid 或 kill -INT `cat /path/pid`   # 快速关闭
pKill -9 主进程pid 或 pkill -9 `cat /path/pid`     # 强制关闭
```

#### nginx重启

```shell
/usr/local/lnmp/nginx-1.13.4/sbin/nginx -s reload
```
同理，重启也能使用kill命令

```shell
kill -USR2 `cat /path/pid`
# 如本文中的示例:
kill -USR2 `cat /usr/local/lnmp/nginx-1.13.4/logs/nginx.pid`
```

### conf
nginx配置文件主要为nginx.conf，这里我们就来详细解读下其中的相关配置指令及含义。  
nginx的强大都是靠配置文件来实现，nginx就是一个二进制文件，nginx读入一个配置文件nginx.conf(nginx.conf可能include包含若干子配置文件)来实现各种各样的功能。我们分段来介绍nginx.conf文件。

#### 全局配置

```shell
worker_processes  1;
#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;
pid        logs/nginx.pid; 
events {
    worker_connections  1024;
}
```
这些是配置文件开始的默认行。通常的环境下，你不需要修改这些选项。这一部分有几个方面需要我们注意：

* 所有以#号开的行是注释，nginx不会解析。默认的配置文件有许多说明解释的注释块
* 指令是以一个变量名开头(例如，worker_processes或pid),然后包含一个参数(例如，1或 logs/nginx.pid)或者多个参数(例如，"logs/error.log notice")
* 所有指令以分号结尾
* 某些指令，像上面的events可以包含多个子指令作为参数。这些子指令以花括号包围。
* 虽然nginx不解析空白符(例如tab，空格，和换行符)，但是良好的缩进能提高你维护长期运行配置文件的效率。良好的缩进使配置文件读起来更流畅，能让你很容易明白配置的策略，即使几个月前。

#### http段
官方定义如下：

```shell
Syntax: http { ... }
Default:    —
Context:    main
```
示例如下： 

```shell
http {
    include       mime.types;
    default_type  application/octet-stream;
​
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
​
    #access_log  logs/access.log  main;
​
    sendfile        on;
    #tcp_nopush     on;
​
    #keepalive_timeout  0;
    keepalive_timeout  65;
​
    #gzip可以参考官方文档：http://nginx.org/en/docs/http/ngx_http_gzip_module.html
    gzip  on;
    gzip_min_length 1000;
    gzip_proxied    expired no-cache no-store private auth;
    gzip_types      text/plain application/xml;
    gzip_comp_level 1;
    include vhost/*.conf;
}
```
> 参考官方文档：[http://nginx.org/en/docs/http/ngx_http_log_module.html#access_log](http://nginx.org/en/docs/http/ngx_http_log_module.html#access_log)

"http { }"块的开头像配置文件的开头一样都是标准配置不需要修改。这里我们需要把注意力放在这些元素上:

 * 这部分内容的开始"include"语句包含/usr/local/lnmp/nginx-1.13.4/conf/mime.types文件到nginx.conf文件include语句所在位置。include对ningx.conf文件的可读性和组织性很有用。
 * 不能过多使用include，如果太多递归地include文件会产生混乱，所以需要合理有限制地使用include来保证配置文件的清晰和可管理。
 * 你可以去掉log_format指令前的注释并修改这几行设置的变量为你想记录的信息。
 * gzip指令告诉nginx使用gzip压缩的方式来降低带宽使用和加快传输速度。如果想使用gzip压缩，需要添加如下配置到配置文件的gzip位置。
 * include vhost/*.conf;表示包含的虚拟主机配置，这将在下一段讲解。

#### server段

虚拟主机配置指令块为server，其包含与http指令块中，为了方面我们配置，我们将其独立出来，通过include指令将其包含进入http指令块中去。

```shell
Syntax: server { ... }
Default:    —
Context:    http
```
示例如下：

```shell
server {
    listen       9091;
    server_name  localhost;
​
    #charset koi8-r;
​
    access_log  logs/host.access.log  main;
​
    location / {
      root   /data0/www/htdocs/lnmp.com;
      index  index.php;
    }
​
    #error_page  404              /404.html;
​
    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
​
    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}
​
    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    location ~ \.php$ {
      root           /data0/www/htdocs/lnmp.com;
      fastcgi_pass   127.0.0.1:9000;
      fastcgi_index  index.php;
      fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
      include        fastcgi_params;
    }
​
    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
```
* **listen**：该配置用于设置nginx监听一个特定的hostname、ip或者端口的连接。默认监听80端口，以下配置都是允许的：

	```shell
	listen     127.0.0.1:80;
	listen     localhost:80;
	listen     12.34.56.79:80;
	```
* **server_name**：该配置可以设置基于域名的虚拟主机，根据请求头中的内容，一个ip的服务器可以配置多个域名，以下配置都是允许的：

	```shell
	server_name lnmp.com www.lnmp.com;
	server_name *.lnmp.com;
	```
	多个域名之间以空格分隔，nginx允许一个虚拟主机有一个或多个名字，也可以使用通配符`*`来设置虚拟主机的名字
* **access_log**：该配置用于配置虚拟主机的日志路径，以下配置均可以，不做赘述：

	```shell
	access_log  logs/lnmp.access.log  main;
	access_log  /data0/www/logs/lnmp.access.log main;
	access_log  off;
	```
* **location**：

	```shell
	语法: location [ = | ~ | ~* | ^~ ] uri { ... }
     	 location @name { ... }
	默认: —
	运行上下文: server, location
	```
	**语法解释**  
	**~** 表示执行一个正则匹配，区分大小写  
	**~\*** 表示执行一个正则匹配，不区分大小写  
	**^~** ^~表示普通字符匹配，如果该选项匹配，只匹配该选项，不匹配别的选项，一般用来匹配目录  
	**=** 进行普通字符精确匹配  
	**@** 定义一个命名的 location，使用在内部定向时，例如 error\_page, try\_files
	
	**loaction分类**：
	1. 普通**location**：（无任何前缀的和`=`，`^~ `，`@`表示普通location）
	2. 正则**location**（`~ `和`~*`前缀表示正则location）

	**location匹配顺序**：  
	正则location匹配让步普通location的严格精确匹配结果；但覆盖普通location的最大前缀匹配结果。具体点就是指：优先普通location匹配中的严格精确匹配，若没有命中严格精确匹配，则进行最长前缀匹配，若命中一个最长前缀匹配，则先暂时定为优先选择，接着进行正则匹配，若正则匹配命中，则使用正则匹配到的结果覆盖之前的最长前缀匹配结果。这里并不是所有的普通location都会进行后续的正则搜索匹配，若最长前缀匹配结果是`^~`和`=`，则会组织后续的正则匹配，直接使用此结果。  
	**location物理位置**：  
	对于location之间的配置顺序，普通location 与其无关，正则location与其有关的。  
更多nginx配置可以查看官方文档：[http://nginx.org/en/docs/](http://nginx.org/en/docs/)

***
# mysql
## 编译安装

```shell
#添加mysql用户组
groupadd mysql
#添加mysql用户，且设置其组为mysql，同时设置其shell为空
useradd -r -g mysql -s /bin/false mysql
#进入源码目录
cd /usr/local/lnmp/src
#解压mysql压缩包
tar -zxf mysql-5.6.37.tar.gz
cd mysql-5.6.37
mkdir bld
cd bld
#编译的参数可以参考:http://dev.mysql.com/doc/refman/5.5/en/source-configuration-option s.html
cmake -DCMAKE_INSTALL_PREFIX=/usr/local/lnmp/mysql-5.6.37 ..
make && make install
chown -R mysql .
chgrp -R mysql .
#初始化数据库，此操作会在安装目录下同时生成my.cnf文件
scripts/mysql_install_db --user=mysql
​
#启动mysql
/usr/local/lnmp/mysql-5.6.37/bin/mysqld --user=mysql --explicit_defaults_for_timestamp
#停掉mysql
/usr/local/lnmp/mysql-5.6.37/bin/mysqladmin shutdown
```
## 配置
>官方文档：[http://dev.mysql.com/doc/refman/5.6/en/server-configuration-defaults.html](http://dev.mysql.com/doc/refman/5.6/en/server-configuration-defaults.html)

mysql启动时，需要加载my.cnf配置文件，加载查找路径为`/etc/my.cnf>$basedir/my.cnf`，本文的查找路径为`/usr/local/lnmp/mysql-5.6.37/my.cnf`，部分配置文件解读示例如下：

```shell
[client]
port = 3306
socket = /tmp/mysql.sock
[mysqld]
port = 3306
socket = /tmp/mysql.sock
basedir = /usr/local/lnmp/mysql-5.6.37
datadir = /usr/local/lnmp/mysql-5.6.37/data
pid-file = /usr/local/lnmp/mysql-5.6.37/data/mysql.pid
user = mysql
bind-address = 0.0.0.0
#表示是本机的序号为1,一般来讲就是master的意思
server-id = 1 
​
#skip-name-resolve
# 禁止MySQL对外部连接进行DNS解析，使用这一选项可以消除MySQL进行DNS解析的时间。但需要注意，如果开启该选项，
# 则所有远程主机连接授权都要使用IP地址方式，否则MySQL将无法正常处理连接请求
#skip-networking
back_log = 600
# MySQL能有的连接数量。当主要MySQL线程在一个很短时间内得到非常多的连接请求，这就起作用，
# 然后主线程花些时间(尽管很短)检查连接并且启动一个新线程。back_log值指出在MySQL暂时停止回答新请求之前的短时间内多少个请求可以被存在堆栈中。
# 如果期望在一个短时间内有很多连接，你需要增加它。也就是说，如果MySQL的连接数据达到max_connections时，新来的请求将会被存在堆栈中，
# 以等待某一连接释放资源，该堆栈的数量即back_log，如果等待连接的数量超过back_log，将不被授予连接资源。
# 另外，这值（back_log）限于您的操作系统对到来的TCP/IP连接的侦听队列的大小。
# 你的操作系统在这个队列大小上有它自己的限制（可以检查你的OS文档找出这个变量的最大值），试图设定back_log高于你的操作系统的限制将是无效的。
max_connections = 1000
# MySQL的最大连接数，如果服务器的并发连接请求量比较大，建议调高此值，以增加并行连接数量，当然这建立在机器能支撑的情况下，因为如果连接数越多，介于MySQL会为每个连接提供连接缓冲区，就会开销越多的内存，所以要适当调整该值，不能盲目提高设值。可以过'conn%'通配符查看当前状态的连接数量，以定夺该值的大小。
max_connect_errors = 6000
# 对于同一主机，如果有超出该参数值个数的中断错误连接，则该主机将被禁止连接。如需对该主机进行解禁，执行：FLUSH HOST。
open_files_limit = 65535
# MySQL打开的文件描述符限制，默认最小1024;当open_files_limit没有被配置的时候，比较max_connections*5和ulimit -n的值，哪个大用哪个，
# 当open_file_limit被配置的时候，比较open_files_limit和max_connections*5的值，哪个大用哪个。
table_open_cache = 128
# MySQL每打开一个表，都会读入一些数据到table_open_cache缓存中，当MySQL在这个缓存中找不到相应信息时，才会去磁盘上读取。默认值64
# 假定系统有200个并发连接，则需将此参数设置为200*N(N为每个连接所需的文件描述符数目)；
# 当把table_open_cache设置为很大时，如果系统处理不了那么多文件描述符，那么就会出现客户端失效，连接不上
max_allowed_packet = 4M
# 接受的数据包大小；增加该变量的值十分安全，这是因为仅当需要时才会分配额外内存。例如，仅当你发出长查询或MySQLd必须返回大的结果行时MySQLd才会分配更多内存。
# 该变量之所以取较小默认值是一种预防措施，以捕获客户端和服务器之间的错误信息包，并确保不会因偶然使用大的信息包而导致内存溢出。
binlog_cache_size = 1M
# 一个事务，在没有提交的时候，产生的日志，记录到Cache中；等到事务提交需要提交的时候，则把日志持久化到磁盘。默认binlog_cache_size大小32K
max_heap_table_size = 8M
# 定义了用户可以创建的内存表(memory table)的大小。这个值用来计算内存表的最大行数值。这个变量支持动态改变
tmp_table_size = 16M
# MySQL的heap（堆积）表缓冲大小。所有联合在一个DML指令内完成，并且大多数联合甚至可以不用临时表即可以完成。
# 大多数临时表是基于内存的(HEAP)表。具有大的记录长度的临时表 (所有列的长度的和)或包含BLOB列的表存储在硬盘上。
# 如果某个内部heap（堆积）表大小超过tmp_table_size，MySQL可以根据需要自动将内存中的heap表改为基于硬盘的MyISAM表。还可以通过设置tmp_table_size选项来增加临时表的大小。也就是说，如果调高该值，MySQL同时将增加heap表的大小，可达到提高联接查询速度的效果
read_buffer_size = 2M
# MySQL读入缓冲区大小。对表进行顺序扫描的请求将分配一个读入缓冲区，MySQL会为它分配一段内存缓冲区。read_buffer_size变量控制这一缓冲区的大小。
# 如果对表的顺序扫描请求非常频繁，并且你认为频繁扫描进行得太慢，可以通过增加该变量值以及内存缓冲区大小提高其性能
read_rnd_buffer_size = 8M
# MySQL的随机读缓冲区大小。当按任意顺序读取行时(例如，按照排序顺序)，将分配一个随机读缓存区。进行排序查询时，
# MySQL会首先扫描一遍该缓冲，以避免磁盘搜索，提高查询速度，如果需要排序大量数据，可适当调高该值。但MySQL会为每个客户连接发放该缓冲空间，所以应尽量适当设置该值，以避免内存开销过大
sort_buffer_size = 8M
# MySQL执行排序使用的缓冲大小。如果想要增加ORDER BY的速度，首先看是否可以让MySQL使用索引而不是额外的排序阶段。
# 如果不能，可以尝试增加sort_buffer_size变量的大小
join_buffer_size = 8M
# 联合查询操作所能使用的缓冲区大小，和sort_buffer_size一样，该参数对应的分配内存也是每连接独享
thread_cache_size = 8
# 这个值（默认8）表示可以重新利用保存在缓存中线程的数量，当断开连接时如果缓存中还有空间，那么客户端的线程将被放到缓存中，
# 如果线程重新被请求，那么请求将从缓存中读取,如果缓存中是空的或者是新的请求，那么这个线程将被重新创建,如果有很多新的线程，
# 增加这个值可以改善系统性能.通过比较Connections和Threads_created状态的变量，可以看到这个变量的作用。(–>表示要调整的值)
# 根据物理内存设置规则如下：
# 1G  —> 8
# 2G  —> 16
# 3G  —> 32
# 大于3G  —> 64
query_cache_size = 8M
#MySQL的查询缓冲大小（从4.0.1开始，MySQL提供了查询缓冲机制）使用查询缓冲，MySQL将SELECT语句和查询结果存放在缓冲区中，
# 今后对于同样的SELECT语句（区分大小写），将直接从缓冲区中读取结果。根据MySQL用户手册，使用查询缓冲最多可以达到238%的效率。
# 通过检查状态值'Qcache_%'，可以知道query_cache_size设置是否合理：如果Qcache_lowmem_prunes的值非常大，则表明经常出现缓冲不够的情况，
# 如果Qcache_hits的值也非常大，则表明查询缓冲使用非常频繁，此时需要增加缓冲大小；如果Qcache_hits的值不大，则表明你的查询重复率很低，
# 这种情况下使用查询缓冲反而会影响效率，那么可以考虑不用查询缓冲。此外，在SELECT语句中加入SQL_NO_CACHE可以明确表示不使用查询缓冲
query_cache_limit = 2M
#指定单个查询能够使用的缓冲区大小，默认1M
key_buffer_size = 4M
#指定用于索引的缓冲区大小，增加它可得到更好处理的索引(对所有读和多重写)，到你能负担得起那样多。如果你使它太大，
# 系统将开始换页并且真的变慢了。对于内存在4GB左右的服务器该参数可设置为384M或512M。通过检查状态值Key_read_requests和Key_reads，
# 可以知道key_buffer_size设置是否合理。比例key_reads/key_read_requests应该尽可能的低，
# 至少是1:100，1:1000更好(上述状态值可以使用SHOW STATUS LIKE 'key_read%'获得)。注意：该参数值设置的过大反而会是服务器整体效率降低
ft_min_word_len = 4
# 分词词汇最小长度，默认4
transaction_isolation = REPEATABLE-READ
# MySQL支持4种事务隔离级别，他们分别是：
# READ-UNCOMMITTED, READ-COMMITTED, REPEATABLE-READ, SERIALIZABLE.
# 如没有指定，MySQL默认采用的是REPEATABLE-READ，ORACLE默认的是READ-COMMITTED
log_bin = mysql-bin
binlog_format = mixed
#超过30天的binlog删除
expire_logs_days = 30 
#错误日志路径
log_error = /usr/local/lnmp/mysql-5.6.37/data/mysql.err
slow_query_log = 1
#慢查询时间 超过1秒则为慢查询
long_query_time = 1 
slow_query_log_file = /usr/local/lnmp/mysql-5.6.37/data/mysql-slow.log
performance_schema = 0
explicit_defaults_for_timestamp
#不区分大小写
#lower_case_table_names = 1 
#MySQL选项以避免外部锁定。该选项默认开启
skip-external-locking 
#默认存储引擎
default-storage-engine = InnoDB 
innodb_file_per_table = 1
# InnoDB为独立表空间模式，每个数据库的每个表都会生成一个数据空间
# 独立表空间优点：
# 1．每个表都有自已独立的表空间。
# 2．每个表的数据和索引都会存在自已的表空间中。
# 3．可以实现单表在不同的数据库中移动。
# 4．空间可以回收（除drop table操作处，表空不能自已回收）
# 缺点：
# 单表增加过大，如超过100G
# 结论：
# 共享表空间在Insert操作上少有优势。其它都没独立表空间表现好。当启用独立表空间时，请合理调整：innodb_open_files
innodb_open_files = 500
# 限制Innodb能打开的表的数据，如果库里的表特别多的情况，请增加这个。这个值默认是300
innodb_buffer_pool_size = 64M
# InnoDB使用一个缓冲池来保存索引和原始数据, 不像MyISAM.
# 这里你设置越大,你在存取表里面数据时所需要的磁盘I/O越少.
# 在一个独立使用的数据库服务器上,你可以设置这个变量到服务器物理内存大小的80%
# 不要设置过大,否则,由于物理内存的竞争可能导致操作系统的换页颠簸.
# 注意在32位系统上你每个进程可能被限制在 2-3.5G 用户层面内存限制,
# 所以不要设置的太高.
innodb_write_io_threads = 4
innodb_read_io_threads = 4
# innodb使用后台线程处理数据页上的读写 I/O(输入输出)请求,根据你的 CPU 核数来更改,默认是4
# 注:这两个参数不支持动态改变,需要把该参数加入到my.cnf里，修改完后重启MySQL服务,允许值的范围从 1-64
innodb_thread_concurrency = 0
# 默认设置为 0,表示不限制并发数，这里推荐设置为0，更好去发挥CPU多核处理能力，提高并发量
innodb_purge_threads = 1
# InnoDB中的清除操作是一类定期回收无用数据的操作。在之前的几个版本中，清除操作是主线程的一部分，这意味着运行时它可能会堵塞其它的数据库操作。
# 从MySQL5.5.X版本开始，该操作运行于独立的线程中,并支持更多的并发数。用户可通过设置innodb_purge_threads配置参数来选择清除操作是否使用单
# 独线程,默认情况下参数设置为0(不使用单独线程),设置为 1 时表示使用单独的清除线程。建议为1
innodb_flush_log_at_trx_commit = 2
# 0：如果innodb_flush_log_at_trx_commit的值为0,log buffer每秒就会被刷写日志文件到磁盘，提交事务的时候不做任何操作（执行是由mysql的master thread线程来执行的。
# 主线程中每秒会将重做日志缓冲写入磁盘的重做日志文件(REDO LOG)中。不论事务是否已经提交）默认的日志文件是ib_logfile0,ib_logfile1
# 1：当设为默认值1的时候，每次提交事务的时候，都会将log buffer刷写到日志。
# 2：如果设为2,每次提交事务都会写日志，但并不会执行刷的操作。每秒定时会刷到日志文件。要注意的是，并不能保证100%每秒一定都会刷到磁盘，这要取决于进程的调度。
# 每次事务提交的时候将数据写入事务日志，而这里的写入仅是调用了文件系统的写入操作，而文件系统是有 缓存的，所以这个写入并不能保证数据已经写入到物理磁盘
# 默认值1是为了保证完整的ACID。当然，你可以将这个配置项设为1以外的值来换取更高的性能，但是在系统崩溃的时候，你将会丢失1秒的数据。
# 设为0的话，mysqld进程崩溃的时候，就会丢失最后1秒的事务。设为2,只有在操作系统崩溃或者断电的时候才会丢失最后1秒的数据。InnoDB在做恢复的时候会忽略这个值。
# 总结
# 设为1当然是最安全的，但性能页是最差的（相对其他两个参数而言，但不是不能接受）。如果对数据一致性和完整性要求不高，完全可以设为2，如果只最求性能，例如高并发写的日志服务器，设为0来获得更高性能
innodb_log_buffer_size = 2M
# 此参数确定些日志文件所用的内存大小，以M为单位。缓冲区更大能提高性能，但意外的故障将会丢失数据。MySQL开发人员建议设置为1－8M之间
innodb_log_file_size = 32M
# 此参数确定数据日志文件的大小，更大的设置可以提高性能，但也会增加恢复故障数据库所需的时间
innodb_log_files_in_group = 3
# 为提高性能，MySQL可以以循环方式将日志文件写到多个文件。推荐设置为3
innodb_max_dirty_pages_pct = 90
# innodb主线程刷新缓存池中的数据，使脏数据比例小于90%
innodb_lock_wait_timeout = 120 
# InnoDB事务在被回滚之前可以等待一个锁定的超时秒数。InnoDB在它自己的锁定表中自动检测事务死锁并且回滚事务。InnoDB用LOCK TABLES语句注意到锁定设置。默认值是50秒
bulk_insert_buffer_size = 8M
# 批量插入缓存大小， 这个参数是针对MyISAM存储引擎来说的。适用于在一次性插入100-1000+条记录时， 提高效率。默认值是8M。可以针对数据量的大小，翻倍增加。
myisam_sort_buffer_size = 8M
# MyISAM设置恢复表之时使用的缓冲区的尺寸，当在REPAIR TABLE或用CREATE INDEX创建索引或ALTER TABLE过程中排序 MyISAM索引分配的缓冲区
myisam_max_sort_file_size = 10G
# 如果临时文件会变得超过索引，不要使用快速排序索引方法来创建一个索引。注释：这个参数以字节的形式给出
myisam_repair_threads = 1
# 如果该值大于1，在Repair by sorting过程中并行创建MyISAM表索引(每个索引在自己的线程内) 
interactive_timeout = 28800
# 服务器关闭交互式连接前等待活动的秒数。交互式客户端定义为在mysql_real_connect()中使用CLIENT_INTERACTIVE选项的客户端。默认值：28800秒（8小时）
wait_timeout = 28800
# 服务器关闭非交互连接之前等待活动的秒数。在线程启动时，根据全局wait_timeout值或全局interactive_timeout值初始化会话wait_timeout值，
# 取决于客户端类型(由mysql_real_connect()的连接选项CLIENT_INTERACTIVE定义)。参数默认值：28800秒（8小时）
# MySQL服务器所支持的最大连接数是有上限的，因为每个连接的建立都会消耗内存，因此我们希望客户端在连接到MySQL Server处理完相应的操作后，
# 应该断开连接并释放占用的内存。如果你的MySQL Server有大量的闲置连接，他们不仅会白白消耗内存，而且如果连接一直在累加而不断开，
# 最终肯定会达到MySQL Server的连接上限数，这会报'too many connections'的错误。对于wait_timeout的值设定，应该根据系统的运行情况来判断。
# 在系统运行一段时间后，可以通过show processlist命令查看当前系统的连接状态，如果发现有大量的sleep状态的连接进程，则说明该参数设置的过大，
# 可以进行适当的调整小些。要同时设置interactive_timeout和wait_timeout才会生效。
[mysqldump]
quick
#服务器发送和接受的最大包长度
max_allowed_packet = 16M 
[myisamchk]
key_buffer_size = 8M
sort_buffer_size = 8M
read_buffer = 4M
write_buffer = 4M
```

## 启动
**mysql**的启动方式主要有三种：  

1. **mysqld_safe启动** ：
	
	```shell
	/usr/local/lnmp/mysql-5.6.37/bin/mysqld_safe --user=mysql
	```
2. **mysqld启动**：

	```shell
	/usr/local/lnmp/mysql-5.6.37/bin/mysqld --user=mysql --explicit_defaults_for_timestamp
	```
3. **init.d启动**：需要将 `support-files/mysql.server`拷贝到`/etc/init.d/mysql.server`

	```shell
	cp support-files/mysql.server /etc/init.d/mysql.server  
	/etc/init.d/mysql.server [start|stop|restart|reload|force-reload|status]
	```
	
***
# PHP
## 编译安装

```shell
#解压源文件
cd /usr/local/lnmp/src
tar -zxf php-5.5.38.tar.gz
​
cd php-5.5.38
#配置编译选项（这里默认编译pdo，fpm，mysql模块，更多编译选项可以通过configure --help 查看）
./configure --prefix=/usr/local/lnmp/php-5.5.38 --enable-fpm --enable-mysqlnd --with-mysql --with-mysqli --with-pdo-mysql
​
make 
#make完成后，会提示进行make test，这一步可以不做，但是建议做一下
make test 
make install
```
## 配置
**php**的配置包括两个部分，一部分是`fpm`的配置，**php**和**nginx**的交互是采用`fpm`的方式进行的；另一部分是`php.ini`，**php**的全局配置。在**php**的源码中就有对应环境的配置，直接拷贝一份即可。本文简单罗列一些基本配置：  

### php.ini
```shell
[PHP]
; 输出缓存允许你甚 在输出正 内容之后发送 header(标头，包括cookies)    ; 或者在这 将指示设为 On  使得所有 件的输出缓存打开。
output_buffering = Off  
​
; 强制flush(刷新)让PHP 告诉输出层在每个输出块之后 动刷新 身数据，建议仅在debug过程中打开。
implicit_flush = Off  
​
; 每个脚本的最 执 时间, 按秒计
max_execution_time = 30 
​
; 个脚本最多可使的内存总大小(这里是8MB)
memory_limit = 1024 
​
[Date]
date.timezone =Asia/Shanghai
​
; E_ALL - 所有的错误和警告  
; E_ERROR - 致命性运 时错  
; E_WARNING - 运 时警告( 致命性错)  
; E_PARSE - 编译时解析错误  
; E_NOTICE - 运 时提醒
; error_reporting = E_ALL & ~E_NOTICE ; 显示所有的错误，除 提醒  
; error_reporting = E_COMPILE_ERROR|E_ERROR|E_CORE_ERROR ; 仅显示错 误 
; 显示所有的错误，除了提醒  
error_reporting = E_ALL & ~E_NOTICE 
; 显示出错误信息(作为输出的一部分)
display_errors = On
log_errors = Off
error_log = logs/error.log
default_mimetype = "text/html"
;default_charset = "iso-8859-1"
; 存放可加载的扩充库(模块)的 录
extension_dir = "./"
;extension=msql.so
; 这条指示告诉PHP是否声明argv和argc变量数 (注:这 argv为数组,argc为变量数)
register_argc_argv=On
​
; PHP将接受的POST数据最大大小。
post_max_size = 8M
​
; 是否允许HTTP方式文件上载
file_uploads = On 
; 存放用HTTP协议上载的文件的临时目录(在没指定时使用系统默认的)
upload_tmp_dir = /tmp
upload_max_filesize = 2M 
[Session]  
; 于保存/取回数据的控制方式   
session.save_handler = files
; 这是数据文件将保存的路径   
session.save_path = /tmp
; 是否使用cookies  
session.use_cookies = 1
session.name = PHPSESSID
; 在请求启动时初始化session
session.auto_start = 0 
; 为按秒记的cookie的保存时间,或为0时,直到浏览器被重启  
session.cookie_lifetime = 0 
; cookie的有效路径  
session.cookie_path = / 
; cookie的有效域
session.cookie_domain = 
; 于连接数据的控制器; php是PHP的标准控制器
session.serialize_handler = php 
; 每个会话初始化启东市启动gc的概率
session.gc_probability = 1 
; 在每次 session 初始化的时候开始的可能性。  
; 在这里数字所指的秒数后，保存的数据将被视为'碎片(garbage)'并由gc进程清理掉
session.gc_maxlifetime = 1440
; 设为{nocache,private,public},以决定HTTP的缓存的问题  
session.cache_limiter = nocache 
; 文档在n分钟后过时
session.cache_expire = 180 
```

### php-fpm.conf
```shell
[global]
pid = run/php-fpm.pid
error_log = log/php-fpm.log
​
; Possible Values: alert, error, warning, notice, debug
log_level = notice
emergency_restart_interval = 0
process_control_timeout = 0
process.max = 128
;process.priority = -19
​
; php-fpm运行模式，默认为后台运行
daemonize = yes
;rlimit_core = 0
​
; fpm使用的事件驱动模型，默认没有设置，进行自动选择
;events.mechanism = epoll
[www]
user = www
group = www
listen = 127.0.0.1:9000
;listen.backlog = 65535
​
; 设置允许链接的客户端IP，默认为任何
;listen.allowed_clients = 127.0.0.1
​
; 子进程控制模式，分为三种：
; static固定模式：子进程数一直等于pm.max_children
; dynamic动态模式：其子进程数量根据max_children、start_servers、min_spare_servers、max_spare_servers决定，但至少会有一个存在
; ondemand按需型：当请求来时才创建，最大存在数取决于max_children，process_idle_timeout指令表示空闲指定时间后退出
pm = dynamic
​
; 最大子进程数
pm.max_children = 5
​
; 启动时创建的子进程数，默认值为min_spare_servers + (max_spare_servers - min_spare_servers) / 2
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
;pm.process_idle_timeout = 500
​
; 处理完多少个请求后重启
;pm.max_requests = 500
​
; 请求日志记录文件
;access.log = log/$pool.access.log
​
; 请求日志文件格式
;access.format = "%R - %u %t \"%m %r%Q%q\" %s %f %{mili}d %{kilo}M %C%%"
​
; 慢日志
;slowlog = log/$pool.log.slow
​
;request_slowlog_timeout = 0
;request_terminate_timeout = 0
​
; 打开文件数，默认为系统上限
;rlimit_files = 1024
```