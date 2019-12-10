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

下面文件展示一个简单的 Nginx模块开发全过程， 我们开发一个叫 echo 的 handler 模块，这个模块功能非常简单， 它接收“echo”指令，指令可指定一个字符串参数，模块会输出这个字符串作为HTTP响应。例如，做如下配置：
```xml
  location /echo {
       echo  "hello nginx";
  }
```
当我们通过浏览器访问 http://hostname/echo时会输出 hello nginx。

直观来看，要实现这个功能需要三步：

- 1、读入配置文件中 echo 指令及其参数。
- 2、进行HTTP包装（添加HTTP头等工作）。
- 3、将结果返回给客户端。

下面本文将分部介绍整个模块的开发过程。

## 2.1、定义模块配置结构

首先我们需要一个结构来存储从配置文件中读进来的相关指令参数———即模块配置信息结构。

根据 Nginx模块开发规则，这个结构的命名规则为 **ngx_http_[module-name]_[main|srv|loc]_conf_t。** 

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

其中两个字段分别表示字符串的长度和数据起始地址。注意在Nginx源代码中对数据类型进行了别称定义，如ngx_int_t为intptr_t的别称，为了保持一致，在开发Nginx模块时也应该使用这些Nginx源码定义的类型而不要使用C原生类型。除了ngx_str_t外，其它三个常用的nginx type分别为：

```c
  typedef intptr_t        ngx_int_t;
  typedef uintptr_t       ngx_uint_t;
  typedef intptr_t        ngx_flag_t;
```

具体定义请参看core/ngx_config.h。关于intptr_t请参考C99中的stdint.h或者[http://linux.die.net/man/3/intptr_t。](http://linux.die.net/man/3/intptr_t)

# 2.2、定义指令
一个 Nginx模块往往接收一至多个指令，echo模块接收一个指令“echo”。

Nginx模块使用一个ngx_command_t数组表示模块所能接收的所有z指令，其中每一个元素表示一个条指令。

ngx_command_t 是ngx_command_s的一个别称（Nginx习惯于使用“_s”后缀命名结构体，然后typedef一个同名“_t”后缀名称作为此结构体的类型名），ngx_command_s定义在core/ngx_config_file.h中：
```C
  struct ngx_command_s {
      ngx_str_t             name;
      ngx_uint_t            type;
      char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
      ngx_uint_t            conf;
      ngx_uint_t            offset;
      void                 *post;
  };
```

- name：指令名称

- type：使用掩码标志位方式配置指令参数，相关可用type定义在core/ngx_config_file.h中：
```C
  #define NGX_CONF_NOARGS      0x00000001
  #define NGX_CONF_TAKE1       0x00000002
  #define NGX_CONF_TAKE2       0x00000004
  #define NGX_CONF_TAKE3       0x00000008
  #define NGX_CONF_TAKE4       0x00000010
  #define NGX_CONF_TAKE5       0x00000020
  #define NGX_CONF_TAKE6       0x00000040
  #define NGX_CONF_TAKE7       0x00000080
 
  #define NGX_CONF_MAX_ARGS    8
 
  #define NGX_CONF_TAKE12      (NGX_CONF_TAKE1|NGX_CONF_TAKE2)
  #define NGX_CONF_TAKE13      (NGX_CONF_TAKE1|NGX_CONF_TAKE3)
 
  #define NGX_CONF_TAKE23      (NGX_CONF_TAKE2|NGX_CONF_TAKE3)
 
  #define NGX_CONF_TAKE123     (NGX_CONF_TAKE1|NGX_CONF_TAKE2|NGX_CONF_TAKE3)
  #define NGX_CONF_TAKE1234    (NGX_CONF_TAKE1|NGX_CONF_TAKE2|NGX_CONF_TAKE3   \
                                |NGX_CONF_TAKE4)
 
  #define NGX_CONF_ARGS_NUMBER 0x000000ff
  #define NGX_CONF_BLOCK       0x00000100
  #define NGX_CONF_FLAG        0x00000200
  #define NGX_CONF_ANY         0x00000400
  #define NGX_CONF_1MORE       0x00000800
  #define NGX_CONF_2MORE       0x00001000
  #define NGX_CONF_MULTI       0x00002000
```
其中 NGX_CONF_NOARGS 表示此指令不接受参数，NGX_CON_F_TAKE1-7 表示精确接收1-7个，NGX_CONF_TAKE12表示接受1或2个参数，NGX_CONF_1MORE表示至少一个参数，NGX_CONF_FLAG表示接受“on|off”……

- set: 一个函数指针，用于指定一个参数转化函数，这个函数一般是将配置文件中相关指令的参数转化成需要的格式并存入配置结构体。
   Nginx 预定义了一些转换函数，可以方便我们调用，这些函数定义在core/ngx_conf_file.h中，一般以“_slot”结尾，
   
   例如ngx_conf_set_flag_slot将“on或off”转换为“1或0”，
   
   再如ngx_conf_set_str_slot将裸字符串转化为ngx_str_t。

- conf: 用于指定 Nginx相应配置文件内存真实地址，一般可以通过内置常量指定，如NGX_HTTP_LOC_CONF_OFFSET。
- offset：指定此条指令的参数的偏移量。

下面是echo模块的指令定义：
```C
  static ngx_command_t  ngx_http_echo_commands[] = {
      {
          ngx_string("echo"),
      
          NGX_HTTP_LOC_CONF_|NGX_CONF_TAKE1,
      
          ngx_http_echo,
      
          offsetof(ngx_http_echo_loc_conf_t, ed),
      
          NULL
      },
      
      ngx_null_command
  };
```
指令数组名命名规则 ngx_http_[module-name]_commands, 注意数组最后一个元素是 ngx_null_command 结束。

参数转化函数的代码如下：
```C
  static char* ngx_http_echo(ngx_cong_t* cf, ngx_command_t* cmd, void* conf) {
     
     ngx_http_core_loc_conf_t *clcf;
     
     clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
     
     clcf->handler = ngx_http_echo_handler;
     
     ngx_conf_set_str_slot(cf,cmd,conf);
     
     return NGX_CONF_OK;
  }
```
这个函数除了调用 ngx_conf_set_str_slot转化echo指令的参数外，还将修改了核心模块配置（也就是这个location的配置），将其handler替换为我们编写的handler：ngx_http_echo_handler。这样就屏蔽了此location的默认handler，使用ngx_http_echo_handler产生HTTP响应。
