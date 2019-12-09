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


每个层级可以有自己的指令(directive)，例如 **worker_processes 是一个main层级指令, 它指定Nginx服务的worker进程数量。** 有的指令只能在一个层级中配置， 如 worker_processes 只能存在于main层中。

而又的指令可以存在于多个层级，在这种情况下，子block会继承父block的配置，同时如果子block配置了与父block不同的指令，则会覆盖掉父block的配置。

**指令的格式是： “参数1  参数2 参数3 ...参数N; ”，** 注意参数间可用任意数量空格分隔，最后要加分号。

在开发Nginx HTTP扩展模块过程中，需要特别注意的是main、server和location三个层级，因为扩展模块通常允许指定新的配置指令在这三个层级中。

最后是配置文件中可以包含配置文件，如上面配置文件中"include mime.types" 就包含了mime.types这个配置文件，此文件指定了各种HTTP Content-type。


一般来说，一个server block表示一个Host， 而里面的一个location则代表一个路由映射规则, 这两个 block 可以说是 HTTP 配置的核心。


下图是Nginx配置文件通常结构图:

![](https://images.cnblogs.com/cnblogs_com/leoo2sk/201104/201104172212324656.png)



## 1.2、Nginx模块工作原理概述

（Nginx本身支持多种模块，如HTTP模块、EVENT模块和MAIL模块，本文只讨论HTTP模块）

Nginx本身做的工作实际很少，当它接到一个HTTP请求时，它仅仅是通过查找配置文件将此次请求映射到一个location block，而此location中所配置的各个指令则会启动不同的模块去完成工作，因此模块可以看做Nginx真正的劳动工作者。

通常一个location中的指令会涉及一个handler模块和多个filter模块（当然，多个location可以复用同一个模块）。

handler模块负责处理请求，完成响应内容的生成，而filter模块对响应内容进行处理。因此Nginx模块开发分为handler开发和filter开发（本文不考虑load-balancer模块）。下图展示了一次常规请求和响应的过程

![](https://images.cnblogs.com/cnblogs_com/leoo2sk/201104/201104171720507387.png)

# 二、Nginx模块开发简单实战

下面文件展示一个简单的Nginx模块开发全过程， 我们开发一个叫 echo 的 handler 模块，这个模块功能非常简单， 它接收“echo”指令，指令可指定一个字符串参数，模块会输出这个字符串作为HTTP响应。例如，做如下配置：
```xml
  location /echo {
       echo  "hello world";
  }
```
当我们通过浏览器访问 http://hostname/echo时会输出 hello nginx。

直观来看，要实现这个功能需要三步：

- 1、读入配置文件中echo指令及其参数。
- 2、进行HTTP包装（添加HTTP头等工作）。
- 3、将结果返回给客户端。

下面本文将分部介绍整个模块的开发过程。

## 2.1、定义模块配置结构

首先我们需要一个结构用于存储从配置文件中读进来的相关指令参数，即模块配置信息结构。

根据Nginx模块开发规则，这个结构的命名规则为 **ngx_http_[module-name]_[main|srv|loc]_conf_t。** 

其中main、srv和loc分别用于表示同一模块在三层block中的配置信息。

这里我们的echo模块只需要运行在loc层级下，需要存储一个字符串参数，因此我们可以定义如下的模块配置：

```c
typedef struct {
   ngx_str_t  ed;
} ngx_http_echo_loc_conf_t;
```

结构体中的ed用来存储echo指令指定的需要输出的字符串。注意这里的ed的类型，在Nginx模块开发中使用ngx_str_t类型表示字符串，这个类型定义在core/ngx_string中：
```c
typedef struct {
    size_t  len;
    
    u_char  *data;
 } ngx_str_t;
```

