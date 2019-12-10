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
# 2.3、创建合并配置信息
下一步是定义模块Context。

这里首先需要定义一个 ngx_http_module_t类型的结构体变量，命名规则为ngx_http_[module-name]_module_ctx，这个结构主要用于定义各个Hook函数。下面是echo模块的context结构：
```C
  static ngx_http_module_t  ngx_http_echo_module_ctx = {
      NULL,                                  /* preconfiguration */
      NULL,                                  /* postconfiguration */
 
      NULL,                                  /* create main configuration */
      NULL,                                  /* init main configuration */
 
      NULL,                                  /* create server configuration */
      NULL,                                  /* merge server configuration */
 
      ngx_http_echo_create_loc_conf,         /* create location configration */
      ngx_http_echo_merge_loc_conf           /* merge location configration */
  };
```
可以看到一共有8个Hook注入点，分别会在不同时刻被Nginx调用，由于我们的模块仅仅用于location域，这里将不需要的注入点设为NULL即可。其中create_loc_conf用于初始化一个配置结构体，如为配置结构体分配内存等工作；merge_loc_conf用于将其父block的配置信息合并到此结构体中，也就是实现配置的继承。这两个函数会被Nginx自动调用。注意这里的命名规则：ngx_http_[module-name]_[create|merge]_[main|srv|loc]_conf。

下面是echo模块这个两个函数的代码：
```C
  static void *
  ngx_http_echo_create_loc_conf(ngx_conf_t *cf)
  {
      ngx_http_echo_loc_conf_t  *conf;
 
      conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_echo_loc_conf_t));
      if (conf == NULL) {
           return NGX_CONF_ERROR;
      }
      conf->ed.len = 0;
      conf->ed.data = NULL;
 
      return conf;
  }
 
  static char *
  ngx_http_echo_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
  {
      ngx_http_echo_loc_conf_t *prev = parent;
      ngx_http_echo_loc_conf_t *conf = child;
 
      ngx_conf_merge_str_value(conf->ed, prev->ed, "");
 
      return NGX_CONF_OK;
  }
```
其中 ngx_pcalloc用于在Nginx内存池中分配一块空间，是pcalloc的一个包装。使用ngx_pcalloc分配的内存空间不必手工free，Nginx会自行管理，在适当是否释放。

create_loc_conf 新建一个 ngx_http_echo_loc_conf_t，分配内存，并初始化其中的数据，然后返回这个结构的指针；merge_loc_conf 将父block域的配置信息合并到 create_loc_conf新建的配置结构体中。

其中 ngx_conf_merge_str_value不是一个函数，而是一个宏，其定义在core/ngx_conf_file.h中：
```C
  #define ngx_conf_merge_str_value(conf, prev, default)                        \
      if (conf.data == NULL) {                                                 \
          if (prev.data) {                                                     \
              conf.len = prev.len;                                             \
              conf.data = prev.data;                                           \
          } else {                                                             \
              conf.len = sizeof(default) - 1;                                  \
              conf.data = (u_char *) default;                                  \
          }                                                                    \
      }
```
同时可以看到，core/ngx_conf_file.h还定义了很多merge value的宏用于merge各种数据。它们的行为比较相似：使用prev填充conf，如果prev的数据为空则使用default填充。

## 2.4、编写Handler

下面的工作是编写handler。handler可以说是模块中真正干活的代码，它主要有以下四项职责：
1. 读入模块配置。
2. 处理功能业务。
3. 生产 http header。
4. 生产 http body。
下面先贴出echo模块的代码，然后通过分析代码的方式介绍如何实现这四步。这一块的代码比较复杂:
```C
static ngx_int_t
ngx_http_echo_handler(ngx_http_request_t *r)
{
    ngx_int_t rc;
    ngx_buf_t *b;
    ngx_chain_t out;
 
    ngx_http_echo_loc_conf_t *elcf;
    elcf = ngx_http_get_module_loc_conf(r, ngx_http_echo_module);
 
    if(!(r->method & (NGX_HTTP_HEAD|NGX_HTTP_GET|NGX_HTTP_POST)))
    {
        return NGX_HTTP_NOT_ALLOWED;
    }
     
    r->headers_out.content_type.len = sizeof("text/html") - 1;
    r->headers_out.content_type.data = (u_char *) "text/html";
 
    r->headers_out.status = NGX_HTTP_OK;
    r->headers_out.content_length_n = elcf->ed.len;
 
    if(r->method == NGX_HTTP_HEAD)
    {
        rc = ngx_http_send_header(r);
        if(rc != NGX_OK)
        {
            return rc;
        }
    }
 
    b = ngx_pcalloc(r->pool, sizeof(ngx_buf_t));
    if(b == NULL)
    {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "Failed to allocate response buffer.");
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }
    out.buf = b;
    out.next = NULL;
 
    b->pos = elcf->ed.data;
    b->last = elcf->ed.data + (elcf->ed.len);
    b->memory = 1;
    b->last_buf = 1;
    rc = ngx_http_send_header(r);
    if(rc != NGX_OK)
    {
        return rc;
    }
    return ngx_http_output_filter(r, &out);
}
```
handler会接收一个ngx_http_request_t指针类型的参数，这个参数指向一个ngx_http_request_t结构体，此结构体存储了这次HTTP请求的一些信息，这个结构定义在http/ngx_http_request.h中：
```C
  struct ngx_http_request_s {
      uint32_t                          signature;         /* "HTTP" */
 
      ngx_connection_t                 *connection;
 
      void                            **ctx;
      void                            **main_conf;
      void                            **srv_conf;
      void                            **loc_conf;
 
      ngx_http_event_handler_pt         read_event_handler;
      ngx_http_event_handler_pt         write_event_handler;
 
  #if (NGX_HTTP_CACHE)
      ngx_http_cache_t                 *cache;
  #endif
 
      ngx_http_upstream_t              *upstream;
      ngx_array_t                      *upstream_states;
                                           /* of ngx_http_upstream_state_t */
 
      ngx_pool_t                       *pool;
      ngx_buf_t                        *header_in;
 
      ngx_http_headers_in_t             headers_in;
      ngx_http_headers_out_t            headers_out;
 
      ngx_http_request_body_t          *request_body;
 
      time_t                            lingering_time;
      time_t                            start_sec;
      ngx_msec_t                        start_msec;
 
      ngx_uint_t                        method;
      ngx_uint_t                        http_version;
 
      ngx_str_t                         request_line;
      ngx_str_t                         uri;
      ngx_str_t                         args;
      ngx_str_t                         exten;
      ngx_str_t                         unparsed_uri;
 
      ngx_str_t                         method_name;
      ngx_str_t                         http_protocol;
 
      ngx_chain_t                      *out;
      ngx_http_request_t               *main;
      ngx_http_request_t               *parent;
      ngx_http_postponed_request_t     *postponed;
      ngx_http_post_subrequest_t       *post_subrequest;
      ngx_http_posted_request_t        *posted_requests;
 
      ngx_http_virtual_names_t         *virtual_names;
 
      ngx_int_t                         phase_handler;
      ngx_http_handler_pt               content_handler;
      ngx_uint_t                        access_code;
 
      ngx_http_variable_value_t        *variables;
     
      /* ... */
  }
```
由于 ngx_http_request_s 定义比较长，这里我只截取了一部分。

可以看到里面有诸如uri，args和request_body等HTTP常用信息。

这里需要特别注意的几个字段是headers_in、headers_out和chain，它们分别表示request header、response header和输出数据缓冲区链表（缓冲区链表是Nginx I/O中的重要内容，后面会单独介绍）。

**第一步: 获取模块配置信息，这一块只要简单使用ngx_http_get_module_loc_conf就可以了。**
```C
#define ngx_http_get_module_main_conf(r, module)                             \
    (r)->main_conf[module.ctx_index]
#define ngx_http_get_module_srv_conf(r, module)  (r)->srv_conf[module.ctx_index]
#define ngx_http_get_module_loc_conf(r, module)  (r)->loc_conf[module.ctx_index]
```

**第二步: 功能逻辑，因为echo模块非常简单，只是简单输出一个字符串，所以这里没有功能逻辑代码。**

**第三步: 设置response header。Header内容可以通过填充headers_out实现，我们这里只设置了Content-type和Content-length等基本内容，ngx_http_headers_out_t 定义了所有可以设置的 HTTP Response Header信息：**
```C
typedef struct {
    ngx_list_t                        headers;
 
    ngx_uint_t                        status;
    ngx_str_t                         status_line;
 
    ngx_table_elt_t                  *server;
    ngx_table_elt_t                  *date;
    ngx_table_elt_t                  *content_length;
    ngx_table_elt_t                  *content_encoding;
    ngx_table_elt_t                  *location;
    ngx_table_elt_t                  *refresh;
    ngx_table_elt_t                  *last_modified;
    ngx_table_elt_t                  *content_range;
    ngx_table_elt_t                  *accept_ranges;
    ngx_table_elt_t                  *www_authenticate;
    ngx_table_elt_t                  *expires;
    ngx_table_elt_t                  *etag;
 
    ngx_str_t                        *override_charset;
 
    size_t                            content_type_len;
    ngx_str_t                         content_type;
    ngx_str_t                         charset;
    u_char                           *content_type_lowcase;
    ngx_uint_t                        content_type_hash;
 
    ngx_array_t                       cache_control;
 
    off_t                             content_length_n;
    time_t                            date_time;
    time_t                            last_modified_time;
} ngx_http_headers_out_t;
```
这里并不包含所有HTTP头信息，如果需要可以使用 agentzh（春来）开发的Nginx模块HttpHeadersMore在指令中指定更多的Header头信息。

设置好头信息后使用 ngx_http_send_heade r就可以将头信息输出，ngx_http_send_header 接受一个 ngx_http_request_t 类型的参数。

**第四步：也是最重要的一步是输出 Response body。这里首先要了解Nginx的I/O机制，Nginx允许handler一次产生一组输出，可以产生多次，Nginx将输出组织成一个单链表结构，链表中的每个节点是一个chain_t，定义在core/ngx_buf.h：**
```C
  struct ngx_chain_s {
      ngx_buf_t    *buf;
      ngx_chain_t  *next;
  };
```
其中ngx_chain_t是ngx_chain_s的别名，buf为某个数据缓冲区的指针，next指向下一个链表节点，可以看到这是一个非常简单的链表。ngx_buf_t的定义比较长而且很复杂，这里就不贴出来了，请自行参考core/ngx_buf.h。ngx_but_t中比较重要的是pos和last，分别表示要缓冲区数据在内存中的起始地址和结尾地址，这里我们将配置中字符串传进去，last_buf是一个位域，设为1表示此缓冲区是链表中最后一个元素，为0表示后面还有元素。因为我们只有一组数据，所以缓冲区链表中只有一个节点，如果需要输入多组数据可将各组数据放入不同缓冲区后插入到链表。下图展示了Nginx缓冲链表的结构：

![](https://images.cnblogs.com/cnblogs_com/leoo2sk/201104/201104191155025791.png)

缓冲数据准备好后，用ngx_http_output_filter就可以输出了（会送到filter进行各种过滤处理）。ngx_http_output_filter的第一个参数为ngx_http_request_t结构，第二个为输出链表的起始地址&out。ngx_http_out_put_filter会遍历链表，输出所有数据。

以上就是handler的所有工作，请对照描述理解上面贴出的handler代码。

## 2.5、组合 Nginx Module
上面完成了Nginx模块各种组件的开发下面就是将这些组合起来了。一个Nginx模块被定义为一个ngx_module_t结构，这个结构的字段很多，不过开头和结尾若干字段一般可以通过Nginx内置的宏去填充，下面是我们echo模块的模块主体定义：
```C
    ngx_module_t  ngx_http_echo_module = {
      NGX_MODULE_V1,
      &ngx_http_echo_module_ctx,             /* module context */
      ngx_http_echo_commands,                /* module directives */
      NGX_HTTP_MODULE,                       /* module type */
      NULL,                                  /* init master */
      NULL,                                  /* init module */
      NULL,                                  /* init process */
      NULL,                                  /* init thread */
      NULL,                                  /* exit thread */
      NULL,                                  /* exit process */
      NULL,                                  /* exit master */
      NGX_MODULE_V1_PADDING
    };
```
开头和结尾分别用NGX_MODULE_V1和NGX_MODULE_V1_PADDING 填充了若干字段，就不去深究了。这里主要需要填入的信息从上到下以依次为context、指令数组、模块类型以及若干特定事件的回调处理函数（不需要可以置为NULL），其中内容还是比较好理解的，注意我们的echo是一个HTTP模块，所以这里类型是NGX_HTTP_MODULE，其它可用类型还有NGX_EVENT_MODULE（事件处理模块）和NGX_MAIL_MODULE（邮件模块）。

这样，整个echo模块就写好了，下面给出echo模块的完整代码：
```C
  /*
   * Copyright (C) Eric Zhang
   */
 
   #include <ngx_config.h>
   #include <ngx_core.h>
   #include <ngx_http.h>
 
  /* Module config */
   typedef struct {
       ngx_str_t  ed;
   } ngx_http_echo_loc_conf_t;
 
   static char *ngx_http_echo(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
   static void *ngx_http_echo_create_loc_conf(ngx_conf_t *cf);
   static char *ngx_http_echo_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child);
 
/* Directives */
   static ngx_command_t  ngx_http_echo_commands[] = {
       { ngx_string("echo"),
         NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
         ngx_http_echo,
         NGX_HTTP_LOC_CONF_OFFSET,
         offsetof(ngx_http_echo_loc_conf_t, ed),
         NULL 
         },
 
         ngx_null_command
    };
 
/* Http context of the module */
static ngx_http_module_t  ngx_http_echo_module_ctx = {
    NULL,                                  /* preconfiguration */
    NULL,                                  /* postconfiguration */
 
    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */
 
    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */
 
    ngx_http_echo_create_loc_conf,         /* create location configration */
    ngx_http_echo_merge_loc_conf           /* merge location configration */
};
 
/* Module */
ngx_module_t  ngx_http_echo_module = {
    NGX_MODULE_V1,
    &ngx_http_echo_module_ctx,             /* module context */
    ngx_http_echo_commands,                /* module directives */
    NGX_HTTP_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};
 
/* Handler function */
static ngx_int_t
ngx_http_echo_handler(ngx_http_request_t *r)
{
    ngx_int_t rc;
    ngx_buf_t *b;
    ngx_chain_t out;
 
    ngx_http_echo_loc_conf_t *elcf;
    elcf = ngx_http_get_module_loc_conf(r, ngx_http_echo_module);
 
    if(!(r->method & (NGX_HTTP_HEAD|NGX_HTTP_GET|NGX_HTTP_POST)))
    {
        return NGX_HTTP_NOT_ALLOWED;
    }
     
    r->headers_out.content_type.len = sizeof("text/html") - 1;
    r->headers_out.content_type.data = (u_char *) "text/html";
 
    r->headers_out.status = NGX_HTTP_OK;
    r->headers_out.content_length_n = elcf->ed.len;
 
    if(r->method == NGX_HTTP_HEAD)
    {
        rc = ngx_http_send_header(r);
        if(rc != NGX_OK)
        {
            return rc;
        }
    }
 
    b = ngx_pcalloc(r->pool, sizeof(ngx_buf_t));
    if(b == NULL)
    {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "Failed to allocate response buffer.");
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }
    out.buf = b;
    out.next = NULL;
 
    b->pos = elcf->ed.data;
    b->last = elcf->ed.data + (elcf->ed.len);
    b->memory = 1;
    b->last_buf = 1;
    rc = ngx_http_send_header(r);
    if(rc != NGX_OK)
    {
        return rc;
    }
    return ngx_http_output_filter(r, &out);
}
 
static char *
ngx_http_echo(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_core_loc_conf_t  *clcf;
    clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
    clcf->handler = ngx_http_echo_handler;
    ngx_conf_set_str_slot(cf,cmd,conf);
     
    return NGX_CONF_OK;
}
 
static void *
ngx_http_echo_create_loc_conf(ngx_conf_t *cf)
{
    ngx_http_echo_loc_conf_t  *conf;
 
    conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_echo_loc_conf_t));
    if (conf == NULL) {
        return NGX_CONF_ERROR;
    }
    conf->ed.len = 0;
    conf->ed.data = NULL;
 
    return conf;
}
 
static char *
ngx_http_echo_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
{
    ngx_http_echo_loc_conf_t *prev = parent;
    ngx_http_echo_loc_conf_t *conf = child;
 
    ngx_conf_merge_str_value(conf->ed, prev->ed, "");
 
    return NGX_CONF_OK;
}
```
