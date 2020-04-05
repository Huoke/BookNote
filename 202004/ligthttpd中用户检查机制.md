# geteuid()和getuid()区别：
- geteuid()：返回有效用户的ID。
- getuid（）：返回实际用户的ID。
## 有效用户ID（EUID）是你最初执行程序时所用的ID  
  表示该ID是程序的所有者  
  真实用户ID（UID）是程序执行过程中采用的ID  
  该ID表明当前运行位置程序的执行者  
## 举个例子  
  程序myprogram的所有者为501/anna  
  以501运行该程序，此时UID和EUID都是501。但是由于中间要访问某些系统资源，需要使用root身份，此时UID为0而EUID仍是501

# lighttpd执行中是必须是以root身份运行的
server.c: main(int argc, char** agrv)
```c
#ifdef HAVE_GETUID
#ifndef HAVE_ISSETUGID
#define issetugid() (geteuid() != getuid() || getuid() != getgid())
#endif
  if (0 != getuid() && issetugid()) { /*在main函数中尽早检查是否是root身份运行程序*/
      fprintf(stderr, "Are you nuts ? Don't apply a SUID bit to this binary\n");
      return -1;
  }
#endif
```
先来看#define issetugid()这个宏函数
