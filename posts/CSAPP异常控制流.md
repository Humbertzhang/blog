# 第八章：异常控制流

#CSAPP

## 1.进程
#### 1.1用户模式和内核模式
用户模式和内核模式的区别在于处理器 psw 的模式位(mode bit)的区别。
用户模式中的进程不允许执行特权指令，也不允许直接引用地址空间中内核区的代码和数据。此时用户程序必须通过*系统调用*（系统调用就是一种trap）来间接地访问内核代码和数据。

进程*从用户模式切换到内核模式的唯一方法*是通过中断、故障或系统调用这种方式。当异常发生时，控制传递到异常处理程序，处理器变为内核模式（改psw），处理程序运行在内核程序中。当返回到应用程序代码时，处理器又会回到用户模式。

#### 1.2上下文切换
内核为每个进程维护了一个上下文，包括各种寄存器信息、内核栈、内核数据结构、页表、进程表、文件表等。
当内核决定调度时，会选择一个新进程，使用上下文切换的方式将程序控制权转移。
一些上下文切换的场景：
* 当内核代表用户执行系统调用时，有可能会发生上下文切换：
如果系统因为等待某个时间发生而阻塞，那么内核可以让当前进程*休眠*（不占用cpu时间），切换到另一个进程。如read，sleep等系统调用就会导致这样的效果。
* 中断也可能导致上下文切换：
比如，所有的系统都有某种产生周期行定时器中断的机制（1ms~10ms）。每次发生定时器中断时，内核就可以判定当前进程已经运行了足够长时间，并切换到一个新进程。

## 2进程控制
#### 2.1.获取进程id
`pid_t getpid（void）` 返回调用该进程的PID。
`pid_t getppid(void)`返回调用该进程的父进程的PID.

#### 2.2.创建和终止进程
进程总是处于以下三种状态：
* 运行：实际包括正在cpu运行(running)和等待被调度到cpu上运行(ready).
* 停止：进程挂起（suspended），不会再被调度。当进程收到 `SIGSTOP、SIGTSTP、SIGTTIN或者SIGOUT`时，进程停止，知道收到`SIGCONT`信号。
* 终止：进程永远地停止。可能原因：
	* 1.收到终止进程信号
	* 2.从主程序返回了
	* 调用了exit函数

*创建：*
通过fork函数。
在fork产生的子进程中通过execve调用其他程序。

fork函数的特性：
* *调用一次，返回两次*：父进程返回子进程pid，子进程返回0.
* *并发执行*：父子进程并发执行
* *相同但是独立的地址空间*：在刚fork完之后，父子进程的地址空间是相同的，但是之后两个进程进行的修改都是独立的。
* 共享文件：子进程继承了父进程打开的文件。

#### 2.3.回收子进程
当一个进程终止后，内核并不是立即清除掉它。该进程会保持在已终止状态，知道被其父进程回收（reaped）。 一个终止了还没回收的进程叫做僵尸进程。
一个父进程死亡时可以把其子进程委托给init进程回收。长时间运行的程序应该总是回收其僵尸子进程，因为他们*仍会消耗系统资源*。


*一个进程可以通过调用waitpid函数来等待其子进程终止或者停止。*

```
pid_t waitpid(pid_t pid, int *statusp, int options)
如果成功则返回子进程PID，如果WNOHANG返回0，其他情况（出错）则返回-1
```

接下来介绍一下这个函数的几个参数的作用：
*pid_t pid*：判定等待集合的成员
* 如果pid>0，则等待集合为一个单独子进程。
* 如果pid=-1，则等待集合为该进程所有子进程。

*int options*：用于修改waitpid的默认行为
options可以为 0(默认情况)，也可以为常量 WNOHANG WUNTRACED WCONTINUED的各种组合。
* 默认：waitpid挂起调用它的进程，直到他的等待集合中的一个子进程终止才返回。如果在其调用时发现已经有子进程终止了，那么他就立刻返回。 返回的都为导致waitpid*已终止*的子进程的PID。
* WNOHANG：等待集合中任何子进程还没终止，则立即返回0。 
用于在等待子进程终止时做一些其他事情。
* WUNTRACED：挂起调用进程的执行，直到等待集合中的一个进程变为已终止或被终止。返回的PID包括*已终止和被停止*的子进程。
* WCONTNIUED：挂起调用进程的执行，知道等待集合中一个进程终止或等待集合中一个被停止的进程收到SIGCONT
* WNOHANG | WUNTRACED：立即返回，如果等待集合中的子进程都没被停止或终止，则返回0，否则返回该子进程的PID。

*int statusp*：用于检查回收的子进程的状态
如果statusp不为空，则waitpid就会在status中存入导致返回的子进程的状态信息。

*waitpid调用错误情况*
如果调用没有子进程，那么waitpid返回-1，设置errno 为 ECHILD。如果waitpid被一个信号中断，则返回-1，设置errno为EINTR。


*wait函数*
`wait(int *statusp) = waitpid(-1, &status, 0)`


## 3.信号
#### 3.1.信号术语

*发送信号*：内核通过更改目的进程上下文中的某个状态，发送一个信号给目的进程。
两种原因可以发送信号：
（1）某个需要发信号的时间发生了，如除零异常或者子进程终止
（2）一个进程调用了kill函数

*接受信号：*
目的进程被*内核*强迫以某种方式对信号的发送作出反应时，他就接收了信号
反应包括：忽略、执行一个信号处理程序(signal handler)

关于信号的其他处理（阻塞等）：
[image:95E1D406-4CBE-4550-B18D-9C19531B5F25-83697-0002FC616765ECA6/E5326B1C-2D75-48E8-B31E-88B1D3A5B08E.png]


#### 3.2.发送信号方式

*3.2.1进程组*
所有发送信号的机制都是基于进程组这个概念的。
每个进程都只属于一个进程组，进程组由进程组ID来标识，默认一个进程和其父进程同属一个进程组，但是可以改变

```
返回调用进程的进程组ID 
pid_t getpgrp(void)
```

```
设置pid的pgid
int setpgid(pid_t pid, pid_t pgid);
```

*3.2.2 使用`/bin/kill/` 发送信号*
usage:
/bin/kill -9 15213
发送信号9(SIGKILL)给进程15213.
/bin/kill -9 -15231
发送信号9(SIGKILL)给进程组15213中的每个进程。

*3.2.3从键盘发送信号*
job: 对一条命令行求值而创建的进程。至多只有一个前台作业（job），但是后台作业可以有很多。

Ctrl + C 会导致内核发送一个SIGINT信号给*前台进程组*中的每个信号。
Ctrl + Z 会导致内核发送一个SIGTSTP信号给*前台进程组*中的每个信号。

*3.2.4用kill函数发信号*
kill函数：
```
int kill(pid_t pid, int sig);
```
pid > 0:kill发送sig给*进程*pid.
pid = 0:kill发送sig给调用进程所在*进程组*的每个进程
pid < 0:kill发送sig给*进程组* |pid| (pid的绝对值)中的每个进程。


*3.2.5 使用alarm函数发送信号*
有点复杂，先不管，530页

#### 3.3接受信号

当内核将进程p从内核模式切换到用户模式时，他会检查进程p的未被阻塞的待处理信号的集合。
如果这个集合为空，那么内核将控制传递到p的逻辑控制流中的下一个指令。
如果非空，那么内核选择某个信号k（通常为最小的k），并且强制p接受信号k。收到这个信号会触发进程采取某种行为，一旦进程完成了这个行为，那么控制就传递会p的下一条指令。

每种信号都有一个预定义的默认行为，包括以下四种：
	* 进程终止
	* 进程终止，转储（存到磁盘上）内存。
	* 进程挂起，直到被SIGCONT信号重启。
	* 进程忽略信号

进程可以通过signal函数*修改某个信号的默认行为*，但是SIGKILL和SIGSTOP的默认行为是不可以修改的。
```
typedef void (*sighandler_t)(int);   //函数指针
sighandler_t signal(int signum, sighandler_t handler);
```

如果handler 为:
	* SIG_IGN，则忽略(ignore)和signum信号相关的行为
	* SIG_DFL，则signum信号恢复默认行为
	* 用户自定义的函数的地址，则进程会在接收到这个信号后调用这个函数。
当处理函数执行完毕后(return)，控制通常传递回控制流中进程被信号接受中断处的指令。


#### 3.4阻塞和解除阻塞信号
隐式阻塞：如果一个进程正在处理某个类型的信号，内核会阻塞这个类型的信号。
显示阻塞：使用sigprocmask函数来明确地阻塞和解除阻塞信号。
```
int sigprocmask(int how, const sigset_t* set, sigset_t* oldset);
关于how:
SIG_BLOCK: 将set中信号添加到blocked中
SIG_UNBLOCK: 从blocked中删除set中的信号
SIG_SETMASK:block = set
关于oldset:
如果oldset为非空指针，则进程的当前信号屏蔽字通过oldset返回。

int sigemptyset(sigset_t* set); // 初始化set为空集合
int sigfillset(sigset_t* set); //把每个信号都添加到set中
int sigaddset(sigset_t* set); //把signum添加到set
int sigdelset(sigset_t* set, int signum); // sigdelset 从set中删除signum

int sigismember(const sigset_t* set, int signum);
// 判断signum是不是set的成员 是返回1，否则返回0
```


#### 3.5编写信号处理程序(sig handler)

*3.5.1 安全的信号处理原则*
编写处理程序的原则：
	* G0:处理程序尽量简单。
	* G1:处理程序中只调用异步信号安全的函数。
	即这个函数要么是可重入的（即每次调用产生的效果一样），要么不可以被信号中断。
	csapp包中提供的简单的异步信号安全的IO函数如下：
```
ssize_t sio_putl(long v);
ssize_t sio_puts(char s[]);

void sio_error(char s[]);
```
	
	* G2:保存和恢复errno
	在处理程序中调用可能会干扰主程序中对他的调用，因此需要：
	在进入处理程序时将errno包存在一个局部变量中，在返回前恢复它.

	* G3:阻塞所有信号，保护对共享全局数据结构的访问。
	如果处理程序和主程序或其他处理程序共享一个全局数据结构，那么在访问这个数据结构时，你的处理程序和主程序都应该阻塞所有信号，以免造成不可预知的结果。

	* G4:使用volatile声明全局变量
	使用volatile的作用是告诉编译器不要缓存这个变量，而是每次从内存中读取。

	* G5:使用sig_atomic_t声明标志（变量）。
	使用 volatile sig_atomic_t 声明的变量的读写会是原子的，但是只适用于单个的读和写，不适用于多条指令的更新。

*3.5.2 正确的信号处理*
信号是不会排队的。如果一个程序正在处理一个类型为A的信号，此时又来了一个信号，会将其放入该进程的类型A的待处理信号中，此后再来类型为A的信号会直接抛弃。

*3.5.3 可移植的信号处理*
为了避免不通OS上signal函数的不一致，Posix定义了标准的sigaction函数，来帮助用户在设置信号处理时，明确地指定他们想要的信号处理语义。
```
int sigaction(int signum, struct sigaction *act, 
struct sigaction *oldact);
```
但是这个并不是很好用，更简单的方式是使用SIgnal，其调用方式与signal一致。
Signal函数设置了一个信号处理程序，其信号处理语义如下：
	* 只有这个处理程序当前正在处理的那种类型的信号被阻塞。
	* 信号不会排队等待。
	* 只要可能，被中断的系统调用会自动重启。
	* 一旦设置了信号处理程序，他就会一致保持，直到Signal带着handler参数为 SIG_IGN 或者 SIG_DFL 被调用。


*3.6 显式地等待信号*
使用 sigsuspend 函数来解决显式等待一个信号的问题。
```
int sigsuspend(const sigset_t *mask);
返回-1
```
sigsuspend函数暂时用mask代替当前的阻塞集合，然后挂起该进程，直到收到一个信号，这个信号对应的行为要么是运行一个信号处理程序，要么是终止进程 才结束。
如果是直接终止进程，那么会直接终止。如果是信号处理程序，那么会返回后恢复*调用之前的阻塞集合*。

*4 非本地跳转*
非本地跳转：将控制从一个函数转移到另一个当前正在执行的函数。非本地跳转可以使用setjmp 和 longjmp实现。
```
sigsetjmp 和 siglongjmp是 setjmp和longjmp的可以被信号处理程序使用的版本.

int setjmp(jmp_buf env);
int sigsetjmp(sigjmp_buf env, int savesigs);

void longjmp(jmp_buf env, int retval);
void siglongjmp(sigjmp_buf env, int retval);
```

setjmp 需要和longjmp配合使用。是一对很有意思的函数：
	* setjmp调用一次返回多次：第一次是当在函数中调用时setjmp时，立即返回0.
	* 第二次是当相应`longjmp`调用时，直接返回`longjmp()`参数中的`retval`
	* longjmp就从不返回，而是在调用时导致setjmp返回。
感觉这对函数有一些goto的意思，但是没有那么灵活。
非本地跳转一个*很重要的应用*就是允许从一个深层嵌套的函数中直接返回，而不需要手动处理函数调用栈（这里也类似goto的一个应用）。


非本地跳转*另一个重要应用*是使一个信号处理程序分支到一个特殊的代码位置。
如下图程序:
[image:5E3A2A67-05B2-4133-A8B8-11AD0B99BBE9-407-00003A5A5A81FAED/EBD429BF-7BE4-421C-BFEA-6B34E1B0DAD2.png]

这个程序的效果就是，在一开始打印
Starting…
之后一直打印 processing….
在Ctrl+ C时，会发给程序SIGINT，程序会调用handler，也就是调用`siglongjmp(buf,1)`，导致`sigsetjmp(buf,1)`返回1，也就导致else条件被触发了，打印restarting….

另外有两点需要注意：
* 为了避免竞争，必须在调用了`sigsetjmp`之后再用Signal设置处理程序。
否则有在`sigsetjmp`为`siglongjmp`设置之前运行处理程序的风险。
* `sigsetjmp`和`siglongjmp`不是异步信号安全函数。因为`siglongjmp`可以跳到任意代码，所以我们必须小心，在`siglongjmp`可达的代码中调用安全的函数。



