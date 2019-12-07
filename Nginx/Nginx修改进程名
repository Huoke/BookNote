# 一、前言
       
       这是一个小插曲... 在ngx_worker_process_cycle()函数里面有个ngx_setproctitle()用来修改worker进程名字。
       
       然后，发现里面的东西很有趣...   关键是里面内容以前我是不知道的，在此记录。


**进程名称在哪儿**


       简单来说，该函数就是用来修改进程名字的。这里参考博文 [《Linux修改进程名称》](https://blog.csdn.net/hengshan/article/details/7835981)，在此感谢博主。

       Linux下用ps命令可以看到显示的进程名字。这个进程的名字会体现在它的main()函数的入参中...

       main函数的原型：

int main(int argc , char *argv[]);
       
    - 不多说，argc是表示命令行参数的个数；
    - argv[]则用于以字符串形式存储所有的命令行参数内容。
       
    OK，Linux中进程的名称就存储在argv[0]中。(这个之前真不了解，囧...)

    然后，Linux还有个环境变量参数信息，表示进程执行需要的所有环境变量信息。它通过一个全局变量Char **environ;来访问环境变量。

    **重点来了：这个argv[]与environ两个变量所占的内存是连续的，并且是environ紧跟在argv[]后面。**

    自己手动验证尝试：
```c
#include <stdio.h>  
#include <string.h>  
 
extern char **environ;  
 
int main(int argc , char *argv[])  
{  
	int i;  
 
	printf("argc:%d\n" , argc);  
 
	for (i = 0; i < argc; i++)
	{  
		printf("argv[%d]:%s\t0x%x\n" , i , argv[i], argv[i]);  
	}  
 
	for (i = 0; i < argc && environ[i]; i++)
	{  
		printf("evriron[%d]:%s\t0x%x\n" , i , environ[i], environ[i]);  
	}  
 
	return 0;  
} 
```

       结果如下：

            



画个简图，比较容易有感觉些吧，看下面：

![](https://img-blog.csdn.net/20140312222004484?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZnp5MDIwMQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


怎么修改进程名称
       

       摘网上方法：

       修改进程名称，只需要修改argv[0]指向的内存的值为所需要的值即可。
       
       但是当我们要修改的值超过argv[0]所指向的内存空间大小时，再这样直接修改，就非常有可能覆盖掉environ的内容，这样就不得了。。从上面的图中，很容易就可以看出。这时候应该满足：

![](https://img-blog.csdn.net/20140312222238500?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZnp5MDIwMQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

必须重新分配一块连续的内存空间，把environ的参数复制到新的空间；无忧修改argv[0]为所需要修改的值；

# 二、好吧，无所谓了。这些个都不是我重点，重点解读Nginx中的源码。。
```c
/* Nginx... */
 
extern char **environ;          // 全局变量，看完上面，你懂得
 
static char *ngx_os_argv_last;  // 指向argv[]最后一个变量
 
/*
 * 先做初始化，完成environ的内容拷贝到新的内存空间
 */
ngx_int_t
ngx_init_setproctitle(ngx_log_t *log)
{
    u_char      *p;
    size_t       size;
    ngx_uint_t   i;
 
    size = 0;
 
	/* 计算environ[]的总大小 */
    for (i = 0; environ[i]; i++) {
        size += ngx_strlen(environ[i]) + 1;
    }
 
	/* 申请同environ大小的内存空间 */
    p = ngx_alloc(size, log);
    if (p == NULL) {
        return NGX_ERROR;
    }
 
	/* 初始化argv[]首尾指针 */
    ngx_os_argv_last = ngx_os_argv[0];
 
	/* 移动argv_last指针到argv[]尾部 */
    for (i = 0; ngx_os_argv[i]; i++) {
        if (ngx_os_argv_last == ngx_os_argv[i]) {
            ngx_os_argv_last = ngx_os_argv[i] + ngx_strlen(ngx_os_argv[i]) + 1;
        }
    }
 
	/* 将environ的内容拷贝到刚申请的内存中去 */
    for (i = 0; environ[i]; i++) {
        if (ngx_os_argv_last == environ[i]) {
 
            size = ngx_strlen(environ[i]) + 1;
            ngx_os_argv_last = environ[i] + size;
 
            ngx_cpystrn(p, (u_char *) environ[i], size);
            environ[i] = (char *) p;  // 这一步就是将environ中的变量重新指向刚分配的内存
            p += size;
        }
    }
 
    ngx_os_argv_last--;
 
    return NGX_OK;
}
```      
      
      完成上面的初始化后，就可以放心修改argv[0]的内容，然后看下具体的ngx_setproctitle()函数。。
      
```c
void
ngx_setproctitle(char *title)
{
    u_char     *p;
 
/* 针对Solaris系统 */
#if (NGX_SOLARIS)
 
    ngx_int_t   i;
    size_t      size;
 
#endif
 
    ngx_os_argv[1] = NULL;
 
	/* 注意ngx_os_argv_last目前指向，如果猜的不错，应该是指向原始environ内存的最后一个参数，
	 * 显然这里的ngx_os_argv_last - ngx_os_argv[0]是原始argv和environ的总体大小
	 */
    p = ngx_cpystrn((u_char *) ngx_os_argv[0], (u_char *) "nginx: ",
                    ngx_os_argv_last - ngx_os_argv[0]);
 
    p = ngx_cpystrn(p, (u_char *) title, ngx_os_argv_last - (char *) p);
 
/* 针对Solaris系统，显示的进程名字方式不一样而已，不看了，原理一样的 */
#if (NGX_SOLARIS) 
 
    size = 0;
 
    for (i = 0; i < ngx_argc; i++) {
        size += ngx_strlen(ngx_argv[i]) + 1;
    }
 
    if (size > (size_t) ((char *) p - ngx_os_argv[0])) {
 
        /*
         * ngx_setproctitle() is too rare operation so we use
         * the non-optimized copies
         */
 
        p = ngx_cpystrn(p, (u_char *) " (", ngx_os_argv_last - (char *) p);
 
        for (i = 0; i < ngx_argc; i++) {
            p = ngx_cpystrn(p, (u_char *) ngx_argv[i],
                            ngx_os_argv_last - (char *) p);
            p = ngx_cpystrn(p, (u_char *) " ", ngx_os_argv_last - (char *) p);
        }
 
        if (*(p - 1) == ' ') {
            *(p - 1) = ')';
        }
    }
 
#endif
	
	/* 在原始argv和environ的连续内存中，将修改了的进程名字之外的内存全部清零 */
    if (ngx_os_argv_last - (char *) p) {
        ngx_memset(p, NGX_SETPROCTITLE_PAD, ngx_os_argv_last - (char *) p);
    }
 
    ngx_log_debug1(NGX_LOG_DEBUG_CORE, ngx_cycle->log, 0,
                   "setproctitle: \"%s\"", ngx_os_argv[0]);
}
```
# 三、总结
       
       这种小东西还是蛮有趣的.
