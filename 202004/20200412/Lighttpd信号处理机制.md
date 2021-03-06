# 一、信号的概念
信号是进程在运行过程中由自身产生或外界环境发送过来的软件中断。可以在头文件signal.h内找到它们的定义（当然，并不是指这些值一定就在signal.h文件内可以看到，而是说在我们的程序里包含头文件signal.h就可以保证程序正确编译，事实上这些宏定义在头文件/usr/include/bits/signum.h内定义），也可以利用命令“kill -l”或“man 7 signal”查看到更详细的说明。

在很多情况下都会有信号的产生：
比如用户按Ctrl+C、Ctrl+Z键时、用户输入kill命令时、硬件异常时、其他进程显示地调用kill函数发送信号时等都将产生信号。

# 二、接收到信号的进程一般有如下三种处理方式：
1. 最简单的处理方式就是忽略接收到的信号。虽然大多数信号都可以使用这种方式处理，但是有两种信号不能被忽略：SIGKILL和SIGSTOP，因为这种信号用来向超级用户提供终止进程的可靠方法。
2. 执行系统默认动作。系统对大多数信号的默认动作就是终止该进程，因此在实际编程时要注意到这一点，以免进入不必要的困境。
3. 捕捉并处理信号。很多时候程序需要对某类信号进行自定义的处理，此时就需要我们捕捉该信号并调用自定义函数进行处理。

# 三、Linux的信号机制
Linux信号机制包括三方面的内容，即信号发送、信号接收、信号处理，如图2-5所示。

![](http://tiebapic.baidu.com/forum/w%3D580/sign=cb2a60956e310a55c424defc87454387/5e350f11728b4710850e07aad4cec3fdfd0323cc.jpg)

图2-5 信号发送、接收与处理

信号的发送时间是任意的，比如运行进程根本就不知道什么时候会收到用户的SIGINT信号（Ctrl+C）。
当运行进程接收到信号后就要对该信号进行处理（我们认为忽略信号也是一种处理）处理不一定立即进行，只有当进程处于用户态时才对信号进行处理，因为如果进程在处于核心态时处理信号，会使得用户代码获取任意权限，即会存在安全隐患。

# 四、Lighttpd中信号处理机制
在Lighttpd由单进程向多进程模型转换前就已经对信号处理函数进行了设置，从而避免了每个子进程也做重复的设置，这当然是可以的，因为当一个进程调用fork时，其子进程将复制父进程的存储映像，因此信号捕捉与处理函数的地址在子进程中也是有意义的，所以子进程继承父进程的信号处理方式。下面，我们来看这段代码（如清单2-4所示）。
```c
#ifdef HAVE_SIGACTION
        struct sigaction act;
	memset(&act, 0, sizeof(act));
	act.sa_handler = SIG_IGN;
	sigaction(SIGPIPE, &act, NULL);
#if defined(SA_SIGINFO)
	act.sa_sigaction = sigaction_handler;
	sigemptyset(&act.sa_mask); // 将信号屏蔽集初始化并清空
	act.sa_flags = SA_SIGINFO; // 设置SA_SIGINF
#else
	act.sa_handler = signal_handler; // 使用字段 sa_handler 设置处理函数
	sigemptyset(&act.sa_mask); // 将信号屏蔽集初始化并清空
	act.sa_flags = 0;
#endif
	sigaction(SIGINT,  &act, NULL); // 设置 SIGINT 信号处理函数
	sigaction(SIGTERM, &act, NULL); // 设置 SIGTERM 信号处理函数
	sigaction(SIGHUP,  &act, NULL); // 设置 SIGHUP 信号处理函数
	sigaction(SIGALRM, &act, NULL); // 设置 SIGALRM 信号处理函数
	sigaction(SIGUSR1, &act, NULL); // 设置 SIGUSR1 信号处理函数

	act.sa_flags |= SA_RESTART | SA_NOCLDSTOP; //在 SIGCHLD 之后重新启动系统应该是安全的
	sigaction(SIGCHLD, &act, NULL);

    /*利用函数 signal()*/ 
#elif defined(HAVE_SIGNAL)
	/* 忽略发送文件中sendfile()的 SIGPIPE */
	signal(SIGPIPE, SIG_IGN);
	signal(SIGALRM, signal_handler);
	signal(SIGTERM, signal_handler);
	signal(SIGHUP,  signal_handler);
	signal(SIGCHLD, signal_handler);
	signal(SIGINT,  signal_handler);
	signal(SIGUSR1, signal_handler);
#endif
...
#ifdef USE_ALARM
	{
		/* 设置周期计时器 (1秒) */
		struct itimerval interval;
		
		interval.it_interval.tv_sec = 1;  // 定时器重启动的间隔值
		interval.it_interval.tv_usec = 0; // 
		interval.it_value.tv_sec = 1;  // 定时器安装后首先启动的初始值
		interval.it_value.tv_usec = 0;
		if (setitimer(ITIMER_REAL, &interval, NULL)) {
			log_error_write(srv, __FILE__, __LINE__, "s", "setting timer failed");
			return -1;
		}
	}
#endif
...
```

相比较signal函数而言， sigaction函数有更多的优势，因此程序利用条件编译宏首先判断是否可以使用sigaction函数，否则使用signal函数。

act是一个sigaction结构体，该结构体定义如下：
```c
struct sigaction {
    void (*sa_handler)(int);
    void (*sa_sigaction)(int, siginfo_t*, void*);
    sigset_t sa_mask;
    int sa_flags;
    void (*sa_restorer)(void);
};
```
- 使用字段sa_handler和sa_sigaction其中的一个（在某些系统上，这两个字段以union的形式出现）用来指定信号处理函数，它们的差别在与设置信号处理函数与信号处理函数之间可传递的信息量不同，字段sa_handler仅接收一个处理信号值的参数，而字段sa_sigaction可接收3个参数，分别为处理信号值、siginfo_t结构体指针变量（需要指定字段sa_flags为SA_SIGINFO）和任意指针变量，因此具体使用哪一个根据需要而定。
- 字段sa_mask用于设置一个信号屏蔽集，在调用信号捕捉处理函数之前，该信号集要加到进程的信号屏蔽字中，只有当从信号捕捉处理函数返回时再将进程的信号屏蔽字恢复为原先值。
- 字段sa_restorer现在不使用。

系统调用getitimer/setitimer用于获取或设置定时器的值，它们的具体含义见表2-7所示。

主题 | 内容
-|-
表头文件 | #include<sys/time.h>
函数定义 | int getitimer(int which, struct itimerval* value); int setitimer(int which, const struct itimerval* value, struct itimerval* ovalue);
函数说明 | 获取或设置定时器的值。 参数which用于指定计时方式，可取值为ITIMER_REAL，ITIMER_VIRTUAL，ITIMER_PROF。其中ITIMER_REAL表示计时真实时间(与alarm类型相同)，当定时器超时时发送SIGALRM信号；ITIMER_VIRTUAL表示计时进程在用户态下的实际执行时间，当定时器超时时发送SIGVTALRM信号；ITIMER_PROF表示计时进程在用户态和核心态下的实际执行时间，当定时器超时时发送SIGPROF信号。itimerval结构体定义为:```c struct itimerval { struct timeval it_interval; // next value struct timeval it_value; // current value};struct timeval {  long tv_sec; // second  long tv_usec; // microseconds};```
返回值 | 执行成功则返回0， 失败返回-1， errno为错误代码。 部分错误代码:  EFAULT：value 或 ovalue不是有效的指针。 EINVAL：which值不是ITIMER_REAL、ITIMER_VIRTUAL或者ITIMER_PROF之一。
附加说明 | 。。。。

信号处理函数设置好后，我们再来看看信号处理函数里面到底做了些什么。很简单，其根据当前进程接收到的信号设置相关变量，虽然仅此而已，但是这些变量值却控制着我们Lighttpd进程运行的全部执行流程。

代码清单2-5信号处理函数
```c
static volatile sig_atomic_t graceful_restart = 0;
static volatile sig_atomic_t graceful_shutdown = 0;
static volatile sig_atomic_t srv_shutdown = 0;
static volatile sig_atomic_t handle_sig_child = 0;
static volatile sig_atomic_t handle_sig_alarm = 1;
static volatile sig_atomic_t handle_sig_hup = 0;
static time_t idle_limit = 0;

#if defined(HAVE_SIGACTION) && defined(SA_SIGINFO)
static volatile siginfo_t last_sigterm_info;
static volatile siginfo_t last_sighup_info;

static void sigaction_handler(int sig, siginfo_t *si, void *context) {
	static const siginfo_t empty_siginfo;
	UNUSED(context);

	if (!si) *(const siginfo_t **)&si = &empty_siginfo;

	switch (sig) {
	case SIGTERM:
		srv_shutdown = 1;
		last_sigterm_info = *si;
		break;
	case SIGUSR1:
		if (!graceful_shutdown) {
			graceful_restart = 1;
			graceful_shutdown = 1;
			last_sigterm_info = *si;
		}
		break;
	case SIGINT:
		if (graceful_shutdown) {
			if (2 == graceful_restart)
				graceful_restart = 1;
			else
				srv_shutdown = 1;
		} else {
			graceful_shutdown = 1;
		}
		last_sigterm_info = *si;

		break;
	case SIGALRM: 
		handle_sig_alarm = 1; 
		break;
	case SIGHUP:
		handle_sig_hup = 1;
		last_sighup_info = *si;
		break;
	case SIGCHLD:
		handle_sig_child = 1;
		break;
	}
}
#elif defined(HAVE_SIGNAL) || defined(HAVE_SIGACTION)
static void signal_handler(int sig) {
	switch (sig) {
	case SIGTERM: srv_shutdown = 1; break;
	case SIGUSR1:
		if (!graceful_shutdown) {
			graceful_restart = 1;
			graceful_shutdown = 1;
		}
		break;
	case SIGINT:
		if (graceful_shutdown) {
			if (2 == graceful_restart)
				graceful_restart = 1;
			else
				srv_shutdown = 1;
		} else {
			graceful_shutdown = 1;
		}
		break;
	case SIGALRM: handle_sig_alarm = 1; break;
	case SIGHUP:  handle_sig_hup = 1; break;
	case SIGCHLD: handle_sig_child = 1; break;
	}
}
#endif
```
sig_atomic_t变量类型值得说明一下，这个类型定义在signal.h头文件（/usr/include/ bits/sigset.h）中：typedef int __sig_atomic_t;
当我们在处理信号（signal）的时候，有时对于一些变量的访问希望不会被中断（无论是来自硬件中断还是软件中断），这就要求访问或改变这些变量需要在计算机的一条指令内完成。通常情况下，int类型的变量通常是原子访问的（另外GUN C的文档也说比int短的类型通常也是具有原子性的，例如short类型，同时指针（地址）类型也一定是原子性的），当我们把变量声明为sig_atomic_t类型时，则会保证该变量在使用或赋值时，无论是在32位还是64位的机器上都能保证操作是原子的。


