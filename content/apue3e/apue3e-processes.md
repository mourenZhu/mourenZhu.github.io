---
title: 'UNIX环境高级编程-进程操作'
date: 2024-11-27T10:26:39+08:00
draft: true
tags: ["unix", "linux", "c"]
author: ["zhumouren"]
---

# 1. 进程环境
```c
#include <stdlib.h>
void exit(int status);
```
exit(0) 与 return 0 是等价的。

# 2. 进程标识和创建子进程
每个进程都有一个非负整型表示的唯一进程ID，但进程ID是可以被复用的。当一个进程终止后进程ID会进入候选池，并且一般新的进程不会使用最近终止进程的进程ID。

```c
#include <unistd.h>

pid_t getpid(void); // 返回值: 调用进程的进程ID

pid_t getppid(void); // 返回值: 调用进程的父进程ID
```
fork() 函数可以创建一个新进程，子进程和父进程继续执行fork调用之后的指令。子进程是父进程的副本。例如，子进程获得父进程数据空间、堆和栈的副本。并且在修改时使用了写时复制（Copy-On-Write，COW）技术。
```c
#include <unistd.h>

pid_t fork(void); // 返回值: 子进程返回0, 父进程返回子进程ID，若出错返回-1
```

以下是一个示例代码，用于创建一个子进程并展示父进程和子进程的ID与父进程ID，以及如何通过信号避免僵尸进程。
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>
#include <sys/wait.h>
#include <sys/types.h>

void sigchld_handler(int sig) {
    int status;
    pid_t id = waitpid(-1, NULL, WNOHANG);
    if (WIFEXITED(status))
    {
        printf("remove proc id: %d \n", id);
        printf("Child send: %d \n", WEXITSTATUS(status));
    }
}

int main(int argc, char *argv[])
{
    struct sigaction sa;
    sa.sa_handler = sigchld_handler;
    sa.sa_flags = SA_RESTART | SA_NOCLDSTOP;

    // 初始化信号屏蔽集，确保不阻塞任何额外信号
    sigemptyset(&sa.sa_mask);

    // 注册 SIGCHLD 信号处理
    sigaction(SIGCHLD, &sa, NULL);

    pid_t pid;
    int val = 24;
    if ((pid = fork()) < 0) {
        fprintf(stderr, "fork err: %d\n", pid);
        exit(1);
    } else if (pid == 0)
    {
        val++;
        printf("child: pid = %d, ppid = %d\n", getpid(), getppid());
        printf("child: val = %d\n", val);
        _exit(0);
    } else 
    {
        sleep(5);
        printf("root: pid = %d, ppid = %d, child_pid = %d\n", getpid(), getppid(), pid);
        printf("root: val = %d\n", val);
        sleep(5);
    }
    
    exit(0);
}
```
以下是执行程序后的结果，一个有意思点是程序的父进程ID是bash，子进程修改的数据并不会影响到父进程的数据。
```
zhumouren@ThinkBook16:~/study_program/apue.3e-ubuntu24/chapter_8$ ./processes_test 
child: val = 25
remove proc id: 174545 
Child send: 0 
root: pid = 174543, ppid = 128279, child_pid = 174545
root: val = 24
zhumouren@ThinkBook16:~/study_program/apue.3e-ubuntu24/chapter_8$ ps
    PID TTY          TIME CMD
  56148 pts/10   00:00:00 bash
  63442 pts/10   00:00:00 ps
```

# 3. 进程终止
进程有5种正常终止以及3种异常终止方式。不管进程如何终止，最后都会执行内核中的同一段代码。这段代码为相应进程关闭所有打开描述符，释放它所使用的存储器等。  
线程终止方式第一遍看起来很枯燥，但很有了解的必要。  

## 3.1 正常终止
1. 在main函数内执行return。等效于调用exit
2. 调用exit()函数，此函数由ISO C定义在<stdlib.h>，其操作会调用各终止处理程序（终止处理程序在调用atexit函数时登记），然后关闭所有标准I/O流等。`适合标准C程序中正常退出使用`  
3. _exit()函数,是一个系统调用在<unistd.h>中,它是低层次的退出函数，不会执行任何清理工作,直接通过内核终止进程，不经过 C 库的清理流程。 `更适合在多线程或 fork 的子进程中使用` 
4. 进程的最后一个线程在其启动例程中执行return语句。但是，该线程的返回值不用作进程的返回值。当最后一个线程从其启动例程返回时，该进程以终止状态0返回。
5. 进程的最后一个线程调用 pthread_exit 函数。如同前面一样，在这种情况中，进程终止状态总是0，这与传送给pthread_exit的参数无关。

## 3.2 异常终止
1. 调用abort。它产生SIGABRT信号，这是下一种异常终止的一种特例。
2. 当进程接收到某些信号时。信号可由进程自身（如调用abort函数）、其他进程或内核产生。如引用错误地址、除以0等。
3. 最后一个线程对“取消”（cancellation）请求作出响应。

# 4. wait()和waitpid()
```c
#include <sys/wait.h>

pid_t wait(int *statloc);
/**
pid: -1 等待任一子进程，>0 等待进程ID与pid相等的子进程，=0 等待组ID等于调用进程组ID的任一子进程，< -1等待组ID等于pid绝对值的任一子进程
*/
pid_t waitpid(pid_t pid, int *statloc, int options);
```
两个函数返回值：若成功，返回进程ID，若出错，返回0（见后面的说明）或−1。这两个函数都可以显示的获取子进程返回值，防止僵尸进程。

两个函数的区别
1. 在一个子进程终止前，wait使其调用者阻塞，而waitpid有一选项，可使调用者不阻塞。
2. waitpid并不等待在其调用之后的第一个终止子进程，它有若干个选项，可以控制它所等待的进程。

以下是wait处理各种返回
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

void
pr_exit(int status)
{
	if (WIFEXITED(status))
		printf("normal termination, exit status = %d\n",
				WEXITSTATUS(status));
	else if (WIFSIGNALED(status))
		printf("abnormal termination, signal number = %d%s\n",
				WTERMSIG(status),
#ifdef	WCOREDUMP
				WCOREDUMP(status) ? " (core file generated)" : "");
#else
				"");
#endif
	else if (WIFSTOPPED(status))
		printf("child stopped, signal number = %d\n",
				WSTOPSIG(status));
}

void
err_sys(const char *msg)
{
    fprintf(stderr, "%s", msg);
    exit(1);
}

int main(int argc, char *argv[])
{
    pid_t	pid;
	int		status;

    if ((pid = fork()) < 0)
		err_sys("fork error");
	else if (pid == 0)				/* child */
		exit(7);

	if (wait(&status) != pid)		/* wait for child */
		err_sys("wait error");
	pr_exit(status);				/* and print its status */

	if ((pid = fork()) < 0)
		err_sys("fork error");
	else if (pid == 0)				/* child */
		abort();					/* generates SIGABRT */

	if (wait(&status) != pid)		/* wait for child */
		err_sys("wait error");
	pr_exit(status);				/* and print its status */

	if ((pid = fork()) < 0)
		err_sys("fork error");
	else if (pid == 0)				/* child */
		status /= 0;				/* divide by 0 generates SIGFPE */

	if (wait(&status) != pid)		/* wait for child */
		err_sys("wait error");
	pr_exit(status);				/* and print its status */

    return 0;
}
```
以下是程序执行的结果。
```
normal termination, exit status = 7
abnormal termination, signal number = 6 (core file generated)
abnormal termination, signal number = 8 (core file generated)
```

# 参考书目
本文章很多内容都是参考《UNIX环境高级编程(第三版)》，基本上就是一个笔记。
1. UNIX环境高级编程(第三版)

