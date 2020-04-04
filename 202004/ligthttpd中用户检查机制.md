server.c: main(int argc, char** agrv)
```c
#ifdef HAVE_GETUID
#ifndef HAVE_ISSETUGID
#define issetugid() (geteuid() != getuid() || getegid() != getgid())
#endif
  if (0 != getuid() && issetugid()) { /*在main函数中尽早检查是否是root身份运行程序*/
      fprintf(stderr, "Are you nuts ? Don't apply a SUID bit to this binary\n");
      return -1;
  }
#endif
```
先来看#define issetugid()这个宏函数
