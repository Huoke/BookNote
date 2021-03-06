# 4.4 设置用户ID和设置组ID
# 4.5 文件访问权限
# 4.6 新文件和目录的所有权
# 4.7 access和faccessat 函数
# 4.8 函数umask

------
# 8.3 fork 函数
一个现有的进程可以调用fork函数创建一个新进程
```c
#include <unistd.h>
pid_t fork(void)

return 子进程返回0，父进程返回子进程ID， 如果出错，返回-1
```
- fork调用一次，返回两次。
- 将子进程ID返回给父进程的原因：因为一个进程的子进程可以有多个，凡是没有一个函数可以获取所有子进程的进程ID，所以将子进程ID返回给父进程。
- 子进程得到返回值0的原因：一个进程只会有一个父进程，所以子进程总是可以调用getppid来获取父进程额进程ID。进程ID=0，是内核交换进程使用，所以一个子进程的进程ID不可能为0。

除了打开文件之外，父进程的很多其他属性也由子进程继承，包括:
- 实际用户ID、实际组ID、有效用户ID、有效组ID
- 附属组ID
- 进程组ID
- 会话ID
- 控制终端
- 设置用户标志和设置组ID标志
- 当前工作目录
- 根目录
- 文件模式创建屏蔽字
- 信号屏蔽和安排
- 对任一打开文件描述符的执行时关闭标志
- 环境
- 连接的共享存储段
- 存储印象
- 资源限制

父进程和子进程之间的区别如下：
- fork的返回值不同
- 进程ID不同
- 这两个进程的父进程ID不同: 子进程的父进程ID是创建它的进程的ID，而父进程的父进程ID则不变
- 子进程的tms_utime、tms_stime、tms_cutime和tms_ustime的值设置为0
- 子进程不继承父进程设置的文件锁
- 子进程的未处理闹钟被清除
- 子进程的未处理信号集设置为空集

使得fork失败的两个主要原因是:
(a) 系统中已经有了太多的进程(通常表明某个方面出了问题)
(b) 该实际用户ID的进程总数超过了系统限制

fork有以下两种用法
(1) 一个父进程希望复制自己，使父进程和子进程同时执行不同的代码段。这在网络服务进程中是常见的————父进程等待客户端的服务请求。当这种请求到达时，父进程调用fork使子进程处理此请求。父进程则继续等待下一个服务请求。
(2) 一个进程要执行一个不同的程序。这对shell是常见的情况。在这种情况下，子进程从frok返回后立即调用exec
# 8.10 函数exec
# 8.11 更改用户ID和更改组ID
