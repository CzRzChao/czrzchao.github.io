---
layout: post
title: "安装lnmp环境"
date: 2017-11-13 00:03:06
description: lnmp环境安装入门
share: true
categories:
 - lnmp
tags:
 - lnmp
---

#安装lnmp环境
##nginx
首先下载源码  
然后安装依赖
	
	yum -y install pcre-devel openssl openssl-devel

配置编译configure

```
./configure --prefix=/usr/local/lnmp/nginx --with-ipv6 --with-http_ssl_module --with-http_realip_module --with-http_addition_module --with-http_dav_module --with-http_flv_module --with-http_mp4_module --with-http_gzip_static_module --with-http_perl_module --with-mail --with-mail_ssl_module
make && make install

```


修改nginx.conf

```
user www www;
worker_processes  auto;

error_log  logs/error.log;

events {
    use epoll;
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    
    keepalive_timeout  65;

    #gzip  on;

    include virtualhost/*.conf;
}
```	
在配置目录下新建*virtualhost*目录  
在*virtualhost*目录下新建配置文件

```
server {
        listen 8800;
        server_tokens off;
        server_name  10.33.106.59;
        access_log  /data0/www/logs/lnmp-access.log;
        error_log   /data0/www/logs/lnmp-error.log;

        root /data0/www/htdocs/lnmp/;
        index index.php;
        if (!-e $request_filename){
                rewrite ^(.*)$ /index.php?_url=$1 last;
        }

        location ~ \.php$ {
                set $real_script_name $fastcgi_script_name;
                if ($fastcgi_script_name ~ /lianjia(/.*)$ ) {
                        set $real_script_name $1;
                }
                fastcgi_pass    127.0.0.1:9999;
                fastcgi_index  index.php;
                fastcgi_param  SCRIPT_FILENAME  $document_root$real_script_name;
                include        fastcgi_params;
        }
    }
```
利用```nginx -t```检测配置语法错误，没问题就可以启动nginx了

##mysql
安装依赖，mysql5.5以后要用	```cmake```进行编译
```
yum -y install gcc gcc-c++ ncurses-devel perl cmake bison
```  
新建mysql安装目录和数据目录  
```
mkdir -p /usr/local/lnmp/mysql5.6
mkdir -p /opt/data/mysql5.6
```   
编译安装   

```
cmake  \
-DCMAKE_INSTALL_PREFIX=/usr/local/lnmp/mysql5.6 \
-DMYSQL_DATADIR=/opt/data/mysql5.6 \
-DSYSCONFDIR=/etc/mysql5.6 \
-DWITH_MYISAM_STORAGE_ENGINE=1 \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_MEMORY_STORAGE_ENGINE=1 \
-DWITH_READLINE=1 \
-DMYSQL_UNIX_ADDR=/tmp/mysql5.6/mysql.sock \
-DMYSQL_TCP_PORT=6000 \
-DENABLED_LOCAL_INFILE=1 \
-DWITH_PARTITION_STORAGE_ENGINE=1 \
-DEXTRA_CHARSETS=all \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci

make & make install
```

创建mysql用户并把更改相关目录归属

```
groupadd mysql
useradd -r -g mysql mysql

chown -R mysql:mysql /usr/local/lnmp/mysql5.6
chown -R mysql:mysql /opt/data/mysql5.6
```

初始化数据库

```
cd /usr/local/lnmp/mysql5.6
./scripts/mysql_install_db --user=mysql --basedir=/usr/local/lnmp/mysql5.6 --datadir=/opt/data/mysql5.6
```
复制mysql服务启动配置文件
```
cp /usr/local/lnmp/mysql5.6/support-files/my-default.cnf /etc/my.cnf
```

拷贝服务脚本到init.d目录
```
cp /usr/local/lnmp/mysql5.6/support-files/mysql.server /etc/init.d/mysqld
```

编辑```/etc/profile```文件

```
vi /etc/profile
```

在文件末尾增加

```
PATH=/usr/local/lnmp/mysql5.6/bin:/usr/local/lnmp/mysql5.6/lib:$PATH
export PATH
```

运行```source /etc/profile```让配置文件生效

运行```service mysqld start```启动

设置mysql开机启动```chkconfig --level 35 mysqld on```

##php
下好源码并解压

编译安装

```
./configure --prefix=/usr/local/lnmp/php --sysconfdir=/usr/local/lnmp/php/etc --with-config-file-path=/usr/local/lnmp/php/lib --enable-fpm --with-mysql=mysqlnd --with-mysqli=mysqlnd --with-pdo-mysql=mysqlnd --with-mhash --with-openssl --with-bz2 --with-curl --with-libxml-dir --with-gd --with-jpeg-dir --with-png-dir --with-zlib --enable-mbstring --with-mcrypt --enable-sockets --with-iconv-dir --with-xsl --enable-zip --with-pcre-dir --with-pear --enable-session --enable-gd-native-ttf --enable-xml --enable-gd-jis-conv --enable-inline-optimization --enable-shared --enable-bcmath --enable-sysvmsg --enable-sysvsem --enable-sysvshm --enable-mbregex --enable-pcntl --with-xmlrpc --with-gettext --enable-exif --with-readline

./configure --prefix=/usr/local/lnmp/php5.5.38 --sysconfdir=/usr/local/lnmp/php5.5.38/etc  --with-config-file-path=/usr/local/lnmp/php5.5.38/lib --enable-fpm --with-mysql=mysqlnd --with-mysqli=mysqlnd --with-pdo-mysql=mysqlnd --with-mhash --with-openssl --with-zlib --with-bz2 --with-curl --with-libxml-dir --with-gd --with-jpeg-dir --with-png-dir --with-zlib --enable-mbstring --with-mcrypt --enable-sockets --with-iconv-dir --with-xsl --enable-zip --with-pcre-dir --with-pear --enable-session  --enable-gd-native-ttf --enable-xml --with-freetype-dir --enable-gd-jis-conv --enable-inline-optimization --enable-shared --enable-bcmath --enable-sysvmsg --enable-sysvsem --enable-sysvshm --enable-mbregex --enable-pcntl --with-xmlrpc --with-gettext --enable-exif --with-readline 

make && make install
```
安装过程中可能有很多依赖，用yum直接安装就好了

中间遇到一个坑，就是lo文件生成失败，最终发现是系统文件```/etc/sysconfig/i18n```文件出问题了
导致lo文件生成失败，该文件是在登录的时候生效的，因此修改后要重新登录才能生效。

安装好后，修改php-fpm.conf监听端口号

```
listen = 127.0.0.1:9999
```
然后将安装包中的php.ini拷到相应目录,修改时区

```
date.timezone = PRC
```
启动php



