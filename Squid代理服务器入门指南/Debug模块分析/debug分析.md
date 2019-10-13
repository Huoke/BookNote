# Debug模块代码分析
这里主要注意一些特殊类型使用和编程技巧
# 一、vs_list
VA_LIST 是在C语言中解决变参问题的一组宏，所在头文件：#include <stdarg.h>
## 成员变量
```C++
#ifdef _M_ALPHA
typedef struct {
char *a0; /* pointer to first homed integer argument */
int offset; /* byte offset of next parameter */
} va_list;
#else
typedef char * va_list;
#endif
```
_M_ALPHA是指DEC ALPHA（Alpha AXP）架构。所以一般情况下va_list所定义变量为字符指针。

## 宏
### INTSIZEOF 宏, 获取类型占用的空间长度，最小占用长度为int的整数倍：
```C++
#define _INTSIZEOF(n) ( (sizeof(n) + sizeof(int) - 1) & ~(sizeof(int) - 1) )
```
关于INTSIZEOF宏更多的解析请查看[c语言不定参数宏INTSIZEOF的由来](https://github.com/Huoke/BookNote/blob/master/Squid%E4%BB%A3%E7%90%86%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%85%A5%E9%97%A8%E6%8C%87%E5%8D%97/Debug%E6%A8%A1%E5%9D%97%E5%88%86%E6%9E%90/c%E8%AF%AD%E8%A8%80%E4%B8%8D%E5%AE%9A%E5%8F%82%E6%95%B0%E5%AE%8FINTSIZEOF%E7%9A%84%E7%94%B1%E6%9D%A5.md)
### VA_START宏，获取可变参数列表的第一个参数的地址（ap是类型为va_list的指针，v是可变参数最左边的参数）：
```C++
#define va_start(ap,v) ( ap = (va_list)&v + _INTSIZEOF(v) )
```
### VA_ARG宏，获取可变参数的当前参数，返回指定类型并将指针指向下一参数（t参数描述了当前参数的类型）：
```C++
#define va_arg(ap,t) ( *(t *)((ap += _INTSIZEOF(t)) - _INTSIZEOF(t)) )
```
### VA_END宏，清空va_list可变参数列表：
```C++
#define va_end(ap) ( ap = (va_list)0 )
```
## 用法
（1）首先在函数里定义一具VA_LIST型的变量，这个变量是指向参数的指针；

（2）然后用VA_START宏初始化刚定义的VA_LIST变量；

（3）然后用VA_ARG返回可变的参数，VA_ARG的第二个参数是你要返回的参数的类型（如果函数有多个可变参数的，依次调用VA_ARG获取各个参数）；

（4）最后用VA_END宏结束可变参数的获取。

## 注意问题
（1）可变参数的类型和个数完全由程序代码控制,它并不能智能地识别不同参数的个数和类型；

（2）如果我们不需要一一详解每个参数，只需要将可变列表拷贝至某个缓冲，可用vsprintf函数；

（3）因为编译器对可变参数的函数的原型检查不够严格,对编程查错不利.不利于我们写出高质量的代码；
