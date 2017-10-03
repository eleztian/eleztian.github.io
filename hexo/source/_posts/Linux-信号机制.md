---
title: Linux 信号机制
date: 2017-10-03 11:32:40
categories: "Linux"
tags: [Linux, c]
keyworlds: Linux, 信号，信号机制
description:
---

# 概念
 
- Linux中的信号（signal)也叫 **“软中断信号”**，用于通知进程发生的异步事件。
- 早期信号的处理方法：
 1. 指定处理函数，通过改函数中断处理 eg. signal(SIGALARM, handler)；
 2. 忽略该信号 eg. signal(SIGALARM, SIG_IGN；
 3. 对该信号的处理保留系统默认（缺省操作），eg. signal(SIGALARM, SIG_DFL),  大部分信号的缺省操作是终止进程；
- 多信号处理
    1. SIG_x 打断 SIG_y的处理函数：
    2. SIG_x 打断 SIG_x的处理函数：
        a. 递归，调用同一个处理函数；
        b. 忽略第二个信号；
        c. 阻塞第二个信号直到第一个信号处理完毕；
- 早期信号处理机制的问题
    1. 捕鼠器问题（不可靠的信号）：
        一个信号表示一个具有破坏性的事情发生并被捕获，当信号或老鼠被捕后，捕鼠器失效，需要重置捕鼠器后才可以继续工作。然而重置捕鼠器是需要一定时间的及时时间很短，这个助理间隙就使得信号处理变得不可靠。
    2. 不知道信号被发送的原因：
        早期模型只是只是告诉处理器，处理函数被调用有什么信号引起，而没有说明产生这个信号的原因。
    3. 处理函数中不能安全的阻塞其他消息：


>在进程表的表项中有一个软中断信号域，该域中每一位对应一个信号，当有信号发送给进程时，对应位置位。由此可以看出，进程对不同的信号可以同时保留，但对于同一个信号，进程并不知道在处理之前来过多少个。



# 编程
- signal.h

1. int signal(int signum, void (*action)(int));
        signum: 信号
        action: 如何响应 （SIG_IGN, SIG_DFL, 函数名（int signum））
        返回 -1 失败

2. int sigaction(int signum, const struct sigaction *action,struct sigaction *prevation)；

---

man 手册查询 man 2 sigaction
```C
        struct sigaction {
            void     (*sa_handler)(int);
            void     (*sa_sigaction)(int, siginfo_t *, void *);
            sigset_t   sa_mask;
            int        sa_flags;
            void     (*sa_restorer)(void);
        };
```
其中 sa_handle 和 sa_sigaction 只能有一个。
sa_sigaction 相比较于sa_handle 可以获取更多的信息通过struct siginfo_t传递，如下：
```C
        struct siginfo_t {
               int      si_signo;    /* Signal number */
               int      si_errno;    /* An errno value */
               int      si_code;     /* Signal code */
               int      si_trapno;   /* Trap number that caused
                                        hardware-generated signal
                                        (unused on most architectures) */
               pid_t    si_pid;      /* Sending process ID */
               uid_t    si_uid;      /* Real user ID of sending process */
               int      si_status;   /* Exit value or signal */
               clock_t  si_utime;    /* User time consumed */
               clock_t  si_stime;    /* System time consumed */
               sigval_t si_value;    /* Signal value */
               int      si_int;      /* POSIX.1b signal */
               void    *si_ptr;      /* POSIX.1b signal */
               int      si_overrun;  /* Timer overrun count; POSIX.1b timers */
               int      si_timerid;  /* Timer ID; POSIX.1b timers */
               void    *si_addr;     /* Memory location which caused fault */
               int      si_band;     /* Band event */
               int      si_fd;       /* File descriptor */
     }

```
sa_mask: 允许设置信号处理函数执行时所需要阻塞的信号。
sa_flags: 修改信号处理函数执行时的默认行为。

>sa_flags specifies a set of flags which modify the behavior of the signal.  It isformed by the bitwise OR of zero or more of the following:

           SA_NOCLDSTOP
                  If signum is SIGCHLD, do not receive notification when child processes
                  stop  (i.e.,  when  they  receive  one of SIGSTOP, SIGTSTP, SIGTTIN or
                  SIGTTOU) or resume (i.e., they receive SIGCONT) (see  wait(2)).   This
                  flag is only meaningful when establishing a handler for SIGCHLD.
           SA_NOCLDWAIT (Since Linux 2.6)
                  If signum is SIGCHLD, do not transform children into zombies when they
                  terminate.  See also waitpid(2).  This flag is  only  meaningful  when
                  establishing a handler for SIGCHLD, or when setting that signal’s dis-
                  position to SIG_DFL.
                  If the SA_NOCLDWAIT flag  is  set  when  establishing  a  handler  for
                  SIGCHLD,  POSIX.1  leaves  it  unspecified whether a SIGCHLD signal is
                  generated when a child process terminates.  On Linux, a SIGCHLD signal
                  is generated in this case; on some other implementations, it is not.
           SA_NODEFER
                  Do not prevent the signal from being received from within its own sig-
                  nal handler.  This flag is only meaningful when establishing a  signal
                  handler.   SA_NOMASK  is  an  obsolete,  non-standard synonym for this
                  flag.
           SA_ONSTACK
                  Call the signal handler on  an  alternate  signal  stack  provided  by
                  sigaltstack(2).   If  an alternate stack is not available, the default
                  stack will be used.  This flag is only meaningful when establishing  a
                  signal handler.
           SA_RESETHAND
                  Restore the signal action to the default state once the signal handler
                  has been called.  This flag is only  meaningful  when  establishing  a
                  signal  handler.   SA_ONESHOT is an obsolete, non-standard synonym for
                  this flag.
           SA_RESTART
                  Provide behavior compatible with BSD signal semantics by  making  cer-
                  tain system calls restartable across signals.  This flag is only mean-
                  ingful when establishing a signal handler.  See signal(7) for  a  dis-
                  cussion of system call restarting.
           SA_SIGINFO (since Linux 2.2)
                  The  signal  handler  takes  3  arguments,  not  one.   In  this case,
                  sa_sigaction should be set instead of sa_handler.  This flag  is  only
                  meaningful when establishing a signal handler.
                  
                 


# 常见问题：
1. 由于信号处理函数是异步执行的，那么我们就无法预知它的执行时间，因此就存在数据共享问题：
    a. 编译器优化导致处理函数中对变量的修改无法让主函数感知到：
```C
    #include <stdio.h>
#include <unistd.h>
#include <signal.h>
#include <string.h>

static int exit_flag = 0;

static void hdl (int sig)
{
    exit_flag = 1;
}

int main (int argc, char *argv[])
{
    struct sigaction act;

    memset (&act, '\0', sizeof(act));
    act.sa_handler = &hdl;
    if (sigaction(SIGTERM, &act, NULL) < 0) {
        perror ("sigaction");
        return 1;
    }

    while (!exit_flag)
        ;

    return 0;
}
```
对上面的代码进行gcc O2级别优化，程序可以正常退出，但是如果采用O3级别优化，那么程序将不能退出。
    因为O2级别的优化时，编译器发现在程序中存在循环读取某一变量时，编译器为了提高效率为将这个变量直接装入寄存器，那么他读取数据时就是直接从寄存器读取而并不是从内存中，这就导致当信号处理函数修改这个变量后，主程序无法感知。解决办法是对变量夹volatile关键字，使其每次都从内存中读取数据。

其他问题后面遇见了再完善。

# 特殊信号处理

> SIGCHLD信号

一般我们不使用这个信号，它的存在作用主要在于清理僵尸进程。
***僵尸进程： 我们知道进程在执行exit()时是释放打开的文件等资源，但在linux中这个进程信息仍在存在进程表中没有被清除，这就是僵尸进程。我们知道进程ID等资源是有限的当僵尸进程过多就会导致无法创建新进程。***
```C
static void sigchld_handle (int sig)
{
    /* 等待所有已经退出的子进程。
     * 这里使用非阻塞的调用以防止子进程在代码其他地方被清理。 */
    while (waitpid(-1, NULL, WNOHANG) > 0) {
    }
}
```
> SIGBUS信号

SIGBUS信号通常是访问被映射的内存时，无法映射到对应文件。***映射（mmap(2)): 内存映射，简而言之就是将用户空间的一段内存区域映射到内核空间，映射成功后，用户对这段内存区域的修改可以直接反映到内核空间，同样，内核空间对这段区域的修改也直接反映用户空间。那么对于内核空间<-->用户空间两者之间需要大量数据传输等操作的话效率是非常高的。***
在这时进程缺省是直接退出，但是我们也是可以对其进行处理的。可以使用sigsetjmp(3)和siglohgjmp(3)跳转到错误的地方，让成宿继续执行。


    DESCRIPTION
       setjmp()  and  longjmp(3) are useful for dealing with errors and inter-
       rupts encountered in a low-level subroutine  of  a  program.   setjmp()
       saves the stack context/environment in env for later use by longjmp(3).
       The stack context will be invalidated  if  the  function  which  called
       setjmp() returns.
       sigsetjmp()  is similar to setjmp().  If, and only if, savesigs is non-
       zero, the process’s current signal mask is saved in  env  and  will  be
       restored if a siglongjmp(3) is later performed with this env.
但是使用它们的时候一定要小心死锁的产生（我并不会用。。。）

> SIGSEGV信号

SIGSEGV 段错误信号（这个倒是常见）, 谁然可以处理但是意义不大。 一种使用场景是在因为写保护而发生SIGEGV时可以通过siginfo_t 得到产出错误的原因从而使用mprotect(2)去除写保护。（没用过）
还有一种是，由于栈空间不足产生，可以使用sigaltstack(2)函数定义独立的栈空间。
> SIGABRT信号

***abort(3)函数:用于终止一个进程。该函数先发送SIGABRT信号，如果该信号被忽略或该信号的处理函数正常返回，它会将信号处理函数重置为默认方式，并重发SIGABRT信号，使进程退出。***
因此处理SIGABRT信号可以在进程结束前做一些事情。
产生SIGABRT的一般原因：

1. 对野指针操作；
2. 对一个地址free()多次；
3. assert失败；
4. 堆越界

# 信号的使用
1. Fork(), Fork()复制一个子进程，这个复制并不会复制父进程的信号队列，子进程会单独创建一个空信号队列。但是子进程会继承父进程所有的信号处理函数和信号阻塞状态。如果父进程以及完成了对信号的设置那么子进程无需再次设置。
2. pthread线程，一个进程下的所有线程有相同的进程ID（PID），向多线程发送信号的情况：
    a. 向进程发送信号：
    b. 向特定线程发送信号：
3. 信号发送

    1. 键盘交互（C+C(SIGINT), C+\(SIGQUIT), C+Z(SIGSTOP)）
    2. kill(pid_t pid, int sig),其中pid:
        - 0:目标为当前进程组的所有进程;
        - -1： 目标是所有的（可以发送信号）进程;
        - <-1: 目标是进程ID为-ID的进程组；
    3. 向进程自身发送信号：raise(3), abort（3）：
        - raise(int sig):可以向进程发送指定信号（在多线程中只能想当前线程发送信号）
        - abort(): 发送SIFABRT信号——>重置该信号->发送SIGABRT信号->程序终止
    4. sigqueue(pid_t pid, int sig, const union sigval value)，与kill类似，多了一个参数可以负载信息，通过siginfo_t获取这个参数。
        
4. 信号阻塞
阻塞信号，加入信号队列。可以防止当前程序被打断，防止信号竞争。实现方式：
    - signal(int signum, sighandler_t handler), 将handler设置为SIG_IGN实现阻塞，该方式已经废弃。
    - sigprocmask(2), 它提供了更多的参数。如下例：
```C
#include <signal.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

static int got_signal = 0;

static void hdl (int sig)
{
    got_signal = 1;
}

int main (int argc, char *argv[])
{
    sigset_t mask;
    sigset_t orig_mask;
    struct sigaction act;

    memset (&act, 0, sizeof(act));
    act.sa_handler = hdl;

    if (sigaction(SIGTERM, &act, 0)) {
        perror ("sigaction");
        return 1;
    }

    sigemptyset (&mask);
    sigaddset (&mask, SIGTERM);

    if (sigprocmask(SIG_BLOCK, &mask, &orig_mask) < 0) {
        perror ("sigprocmask");
        return 1;
    }

    sleep (10);

    if (sigprocmask(SIG_SETMASK, &orig_mask, NULL) < 0) {
        perror ("sigprocmask");
        return 1;
    }

    sleep (1);

    if (got_signal)
        puts ("Got signal");

    return 0;
}
```



