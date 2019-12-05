# nginx源码分析之模块初始化
在nginx启动过程中，模块的初始化是整个启动过程中的重要部分，而且了解了模块初始化的过程对应后面具体分析各个模块会有事半功倍的效果。在我看来，分析源码来了解模块的初始化是最直接不过的了，所以下面主要通过结合源码来分析模块的初始化过程。

稍微了解nginx的人都知道nginx是高度模块化的，各个功能都封装在模块中，而各个模块的初始化则是根据配置文件来进行的，下面我们会看到nginx边解析配置文件中的指令，边初始化指令所属的模块，指令其实就是指示怎样初始化模块的。

## 一、模块初始化框架
模块的初始化主要在函数ngx_init_cycle（src/core/ngx_cycle.c）中下列代码完成：
```C
ngx_cycle_t* ngx_init_cycle(ngx_cycle_t *old_cycle)
{
    ...
    //配置上下文
    cycle->conf_ctx = ngx_pcalloc(pool, ngx_max_module * sizeof(void *));
    ...
    //处理core模块，cycle->conf_ctx用于存放所有CORE模块的配置
    for (i = 0; ngx_modules[i]; i++) {
        if (ngx_modules[i]->type != NGX_CORE_MODULE) {  //跳过不是nginx的内核模块
            continue;
        }
        module = ngx_modules[i]->ctx;
        //只有ngx_core_module 有create_conf 回调函数,这个会调用函数会创建 ngx_core_conf_t结构，
        //用于存储整个配置文件 main scope范围内的信息，比如 worker_processes，worker_cpu_affinity等
        if (module->create_conf) {
            rv = module->create_conf(cycle);
            ...
            cycle->conf_ctx[ngx_modules[i]->index] = rv;
        }
    }
    //conf表示当前解析到的配置命令上下文，包括命令，命令参数等
    conf.args = ngx_array_create(pool, 10, sizeof(ngx_str_t));
    ...
    conf.temp_pool = ngx_create_pool(NGX_CYCLE_POOL_SIZE, log);
    ...
    conf.ctx = cycle->conf_ctx;
    conf.cycle = cycle;
    conf.pool = pool;
    conf.log = log;
    conf.module_type = NGX_CORE_MODULE;  //conf.module_type指示将要解析这个类型模块的指令
    conf.cmd_type = NGX_MAIN_CONF;  //conf.cmd_type指示将要解析的指令的类型
    //真正开始解析配置文件中的每个命令
    if (ngx_conf_parse(&conf, &cycle->conf_file) != NGX_CONF_OK) {
     ...
    }
    ...
    //初始化所有core module模块的config结构。调用ngx_core_module_t的init_conf,
    //在所有core module中，只有ngx_core_module有init_conf回调，
    //用于对ngx_core_conf_t中没有配置的字段设置默认值
    for (i = 0; ngx_modules[i]; i++) {
        if (ngx_modules[i]->type != NGX_CORE_MODULE) {
            continue;
        }
        module = ngx_modules[i]->ctx;
        if (module->init_conf) {
            if (module->init_conf(cycle, cycle->conf_ctx[ngx_modules[i]->index])
                == NGX_CONF_ERROR)
            {
                environ = senv;
                ngx_destroy_cycle_pools(&conf);
                return NULL;
            }
        }
    }
    ...
}
```
