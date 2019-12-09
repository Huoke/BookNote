# 一、Nginx 的配置
## 1.1、Nginx的配置基本结构

配置文件可以看做是Nginx的灵魂，Nginx服务在启动时会读入配置文件，后续的一切动作行为都是按照配置文件中的指令进行的，所以如果将Nginx本身看做一个计算机，那么Nginx的配置文件可以看成是全部的程序指令。

下面是一个Nginx配置文件的实例：
```xml
 #user  nobody;
 worker_processes  8;
 error_log  logs/error.log;
 pid        logs/nginx.pid;
 events {
     worker_connections  1024;
 }
 http {
     include       mime.types;
     default_type  application/octet-stream;
     sendfile        on;
     #tcp_nopush     on;
     keepalive_timeout  65;
     #gzip  on;
     server {
         listen       80;
         server_name  localhost;
         location / {
             root   /home/yefeng/www;
             index  index.html index.htm;
         }
         #error_page  404              /404.html;
         # redirect server error pages to the static page /50x.html
         #
         error_page   500 502 503 504  /50x.html;
         location = /50x.html {
             root   html;
         }
     }
 }
```
Nginx 配置文件时纯文本，你可以用任何编辑器如vim或emacs打开它，通常它会在nginx安装目录的conf下， 如果我的nginx安装在/usr/local/nginx, 主要配置默认放在/usr/local/nginx/conf/nginx.conf。

其中“#”表示此行是注释。

Nginx的配置文件是以block(块)的形式组织的，一个block通常使用大括号"{}"表示。block分为几个层级，整个配置文件为main层， 这是最大的层级。

在main层级下可以有event、http等层级。

在http层中又会有server block ； 在server block中可以包含location block。

