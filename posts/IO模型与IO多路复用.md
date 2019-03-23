<!-- TOC -->
- [1. I/O模型](#1-io模型)
    - [1.1 五种 I/O模型](#11-五种-io模型)
    - [1.2 五种模型比较](#12-五种模型比较)
- [2. I/O多路复用](#2-io多路复用)
    - [2.1 select](#21-select)
        - [2.1.1 函数原型](#211-函数原型)
        - [2.1.2 原理](#212-原理)
    - [2.2 poll](#22-poll)
        - [2.2.1函数原型](#221函数原型)
        - [2.2.2 原理](#222-原理)
    - [2.3 epoll](#23-epoll)
        - [2.3.1函数原型](#231函数原型)
        - [2.3.2原理](#232原理)
        - [2.3.3 解决了什么问题？如何解决的？](#233-解决了什么问题如何解决的)
    - [2.4 kqueue](#24-kqueue)
    - [2.5 各种I/O多路复用对比](#25-各种io多路复用对比)

<!-- /TOC -->


# 1. I/O模型

一次I/O操作包括两个阶段：

- 等待数据准备好
- 从内核向进程复制数据



### 1.1 五种 I/O模型

- 阻塞式I/O
- 非阻塞式I/O
- I/O多路复用
- 信号驱动式I/O
- 异步I/O



recvfrom 系统调用：用于接受Socket传来的数据，并将从内核复制到应用程序的buf中。



1. 阻塞式I/O： 调用了 recvfrom 后，进程阻塞在该函数上，等待数据准备好，并从内核向进程复制完数据后返回。

 ![img](https://gitee.com/CyC2018/CS-Notes/raw/master/docs/pics/1492928416812_4.png)

​						 

1. 非阻塞式I/O： 调用了 recvfrom 后，进程不阻塞在该函数上，而是一直调用此函数询问是否准备好了数据（轮训），等准备好了数据后，再从内核向进程复制数据，之后返回成功。

![img](https://gitee.com/CyC2018/CS-Notes/raw/master/docs/pics/1492929000361_5.png)



1. I/O多路复用：使用select （或poll epoll）来等待多个套接字。当其中的任意一个变为可读时，返回 select（或poll epoll），之后再使用recvfrom函数将数据从内核复制到进程中。实现了用一个线程来检查多个描述符的就绪状态。

![img](https://gitee.com/CyC2018/CS-Notes/raw/master/docs/pics/1492929444818_6.png)

1. 信号驱动I/O：进程一开始调用信号函数（用于通知内核），信号函数直接返回。当**内核将数据准备好了**之后，向应用进程发送信号。之后应用进程使用recvfrom()系统调用从内核中复制信息到进程中。

   即在等待数据准备好期间是非阻塞的。

![img](https://gitee.com/CyC2018/CS-Notes/raw/master/docs/pics/1492929553651_7.png)

1. 异步I/O：进程开始调用aio_read()，其立即返回，应用程序继续执行而不会被阻塞。当内核完成了：**准备好数据 + 将数据从内核复制到应用进程** 两步后，再用信号通知进程。

![img](https://gitee.com/CyC2018/CS-Notes/raw/master/docs/pics/1492930243286_8.png)





### 1.2 五种模型比较

![img](https://gitee.com/CyC2018/CS-Notes/raw/master/docs/pics/1492928105791_3.png)



前四种为同步操作，只有异步I/O为异步操作。

| I/O类别     | 第一阶段操作                                                 | 第一阶段是否阻塞 | 第二阶段操作                                                 | 第二阶段是否阻塞 |
| ----------- | ------------------------------------------------------------ | ---------------- | ------------------------------------------------------------ | ---------------- |
| 阻塞I/O     | 调用recvfrom()后开始阻塞                                     | 阻塞             | 在阻塞状态下，等待数据准备好之后将数据从内核复制到进程。     | 阻塞             |
| 非阻塞I/O   | 持续调用recvfrom()，直到数据准备好。                         | 非阻塞           | 再次调用recvfrom()，将数据从内核复制到进程中。               | 阻塞             |
| I/O复用     | 调用select(或poll epoll)，等待多个套接字中有任意一个准备好。直到select(或poll epoll 返回) | 阻塞             | 调用recvfrom()，将相应套接字对应的数据从内核复制到进程。     | 阻塞             |
| 信号驱动I/O | 进程调用sigaction系统调用，立即返回。进程继续执行。          | 非阻塞           | 内核等待数据准备好后，使用信号通知进程，进程收到信号后将数据从内核复制到进程。 | 阻塞             |
| 异步I/O     | 进程调用aio_read系统调用，立即返回。                         | 非阻塞           | 内核在数据准备好后将数据从内核复制到相应进程。               | 非阻塞           |



# 2. I/O多路复用



### 2.1 select 

#### 2.1.1 函数原型

```c
int select(int maxfdp1, fd_set *readset, fd_set *writeset, fd_set *exceptset, const struct timeval *timeout)
```

其中各参数介绍如下：

`返回值`：就绪的描述符的数目。

`maxfdp1`: 指定待测试的描述字个数，其值为待测试的最大描述字+1（因此叫maxfdp1），描述字0,1,2 ... maxfdp1-1 都将被测试。

`readset` `writeset` `exceptset` :我们指定的需要内核测试读、写、异常的描述字。可为空，即不检查此条件。

`fd_set`为一个集合，这个集合中存放的是文件描述符，可由以下四个宏设置：

- void FD_ZERO(fd_set *fdset);  // 清空集合
- void FD_SET(int fd, fd_set* fdset); // 将给定的文件描述符加入到集合中
- void FD_CLR(int fd, fd_set* fdset); // 将给定的文件描述符从集合中删除
- int FD_ISSET(int fd, fd_set* fdset); // 检查集合中指定的文件描述符是否可以读写。

`timeout`：用于指定超时时间。

```c
struct timeval {
    long tv_sec; // 秒
    long tv_usec; // 毫秒
}
```

这个参数有三种可能：

1. 永远等待： 仅在有描述符准备好时才返回。此时需设为NULL。
2. 等待一段固定时间：在有一个描述符准备好时返回，但不超过该设定的时间。
3. 直接返回（轮训）：将值设为0。

#### 2.1.2 原理

select函数监视3类描述符，调用select后会阻塞，直到这三个中有描述符就绪或超时。当select函数返回后，需要遍历fdset 来找到是哪个描述符就绪。



### 2.2 poll

#### 2.2.1函数原型

```c
int poll(struct pollfd* fds, unsigned int nfds, int timeout);
```

每个pollfd结构体指定了一个被监视的文件描述符，结构如下：

```c
struct pollfd {
    int fd; 	   //文件描述符
    short events;  // 对文件描述符上感兴趣的事件
    short revents; // fd上当前实际发生的事件
}
```



`返回值`： fds中读、写、出错的描述符数量。返回0时表示超时，返回-1表示出错。

`fds`：一个pollfd数组，存放需要检测的socket描述符，调用poll()后fds数组不会被清空。

`nfds`: 记录数组fds中描述符的总数量。

`timeout`: 超时时间，单位毫秒。

#### 2.2.2 原理

与select相同，每次调用poll时，都需要遍历全部的文件查看是否准备好，也需要把fd的集合从用户态拷贝到内核态，但是poll的最大连接数没有限制(因为底层使用了**链表**)，如果空间允许的话，可以加入文件描述符，但是过多的文件描述符还是会降低返回速度。



### 2.3 epoll

epoll 是 select和poll的增强版本。

相对于select和poll来说，epoll更加灵活，没有描述符限制。epoll使用一个文件描述符管理多个描述符，将用户想要监听的文件描述符以及配套的事件 (即下文中的`epitem`结构体) 存放到内核中，这样在用户空间和内核空间的copy只需一次。

此外，还通过**回调**的方式解决了每次调用都要遍历所有文件描述符的问题。

这些改进使得epoll能**高效支持百万级别**的描述符监听。



#### 2.3.1函数原型

```c
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event * event);
int epoll_wait(int epfd, struct epoll_events * events, int maxevents, int timeout);
```

- int epoll_create(int size)`

  创建一个epoll句柄，size用于告诉内核该监听的数目有多大。

- `int epoll_ctl(int epfd, int op, int fd, struct epoll_event * event);`

  epoll的事件注册函数。用来注册这个epoll要监听什么事件。

  `epfd`: epoll_create的返回值.

  `op`：动作，包括注册新fd到epfd中；修改已注册的fd的监听事件；从epfd中删除一个fd；

  `fd`: 需要监听的fd

  `event`：告诉内核对该fd需要监听什么事件 的结构体。

- `int epoll_wait(int epfd, struct epoll_events * events, int maxevents, int timeout);`

  epoll_wait函数调用后开始等待事件的就绪，成功时返回就绪的事件数目。

  `epfd`:epoll句柄

  `events`: 就绪事件的集合

  `maxevents`: 最多event的数目

  `timeout`:等待的超时时间



epoll有两种工作模式：水平触发和边缘触发。

**水平触发**(LT level trigger)：默认工作模式，当epoll_wait检测到某事件就绪并通知应用程序时，应用程序可以不立即处理该事件，下次调用epoll_wait时会再次通知此事件。

**边缘触发**(ET edge trigger)：当epoll_wait检测到某事件就绪并通知应用程序时，应用程序必须立即处理。若不处理下次调用也不会通知此事件了。

边缘触发模式在很大程度上减少了epoll事件被重复触发的次数，因此效率比水平触发模式要高。



#### 2.3.2原理

当某进程**调用epoll_create方法**时，Linux内核会创建一个eventpoll结构体.

```c
struct eventpoll {
    ...
    struct rb_root 	 rbr;    // 红黑树的根结点，这颗树中存储着所有添加到epoll中的需要监控的事件
    struct list_head rdlist; // 双向链表，存放着将要通过event_wait返回给用户的满足条件的事件
    ...
}
```

红黑树和双向链表中的结构体都包含以下结构体来记录必要信息：

```c
struct epitem {
    struct rb_node  rbn;//红黑树节点
    struct list_head    rdllink;//双向链表节点
    struct epoll_filefd  ffd;  //事件句柄信息
    struct eventpoll *ep;    //指向其所属的eventpoll对象
    struct epoll_event event; //期待发生的事件类型
}
```

每一个epoll对象都有一个独立的eventpoll结构体，**调用event_ctl添加事件**时，这些事件都会以红黑树的节点的形式挂载在红黑树中，这样重复添加的事件就可以通过红黑树高效地识别出来（红黑树插入时间效率为log(n)，n为树高度）。

所有添加到epoll中的事件都会在内核中断处理程序中设置一个回调方法(`ep_poll_callback`)，当相应事件发生时，内核便调用这个回调方法。这个回调方法会将发生的事件添加到rdlist双链表中，表示该事件已就绪。

![](https://ae01.alicdn.com/kf/HTB1CY6ENwHqK1RjSZFEq6AGMXXaz.jpg)



**当进程调用epoll_wiat进行检查时**，只需要查看rdlist是否为空，如果为空则只需要继续sleep直到不为空时返回这个rdlist中的数据即可。 



#### 2.3.3 解决了什么问题？如何解决的？

select的缺点：

- 每次调用select都需要将所有fd集合从用户态拷贝到内核态，当fd很多时开销很大。
- 每次调用select都需要在内核遍历传进来的所有fd，开销也会很大。
- select支持的文件描述符较小，默认1024

poll: 虽然通过使用链表避免了最多的文件描述符的限制，但是以下两个缺点还是存在。

- 每次调用都需要将fd集合从用户态拷贝到内核态
- 每次调用都需要在内核遍历传进来的所有fd    



而epoll则解决了这些问题。

1. 问题一：每次调用需要传所有fd（文件描述符）集合到内核

   我们在调用epoll_wait时就相当于之前调用select/poll，但是此时我们却不用传递所有的文件描述符了，因为我们已经在调用epoll_ctl时就将文件描述符传给了内核，并且这些被传递的文件描述符只需要被epoll_ctl传递，之后在每次调用epoll_wait时都无需再次传递。

2. 问题二：每次调用都需要遍历所有fd

   通过操作系统级别支持的回调解决了这个问题。

   当注册在红黑树中的epitem所监听的事件发生时，内核会调用回调函数`ep_poll_callbakc` 将其加入到rdlist中。 当进程调用epoll_wiat进行检查时，只检查rdlist是否为空，如果为空则只需要继续sleep直到不为空时返回这个rdlist中的数据即可。 



如此，一颗红黑树，一张准备就绪句柄链表，少量的内核cache，就帮我们解决了大并发下的socket处理问题。 

在执行epoll_create时，创建了红黑树和就绪链表。

在执行epoll_ctl时，如果增加socket文件描述符，则检查在红黑树中是否存在，存在立即返回，不存在则添加到树干上，然后向内核注册**回调**函数，用于当中断事件来临时向准备就绪链表中插入数据。

 执行epoll_wait时立刻返回准备就绪链表里的数据即可。



// TODO：kqueue与epoll相似，也是基于回调来实现的。是在Unix中的实现，暂时先不总结了

### 2.4 kqueue

函数原型

原理



### 2.5 各种I/O多路复用对比

| 类别   | 操作方式 | 数据结构    | 效率                                                         | 最大连接数 | fd拷贝方式                                                   | 优点                                                         | 缺点                                                         | 适用场景                                               |
| ------ | -------- | ----------- | ------------------------------------------------------------ | ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------ |
| select | 遍历     | 数组        | 每次调用都线性遍历所有fd集合。时间复杂度O(n)。               | 1024       | 每次调用，都需要把fd集合从用户态拷贝到内核态。               | 1.select的timeout参数精度为1ns,poll 和epoll 为1ms。<br />2. select移植性好，几乎被所有主流平台支持。 | 1.最大描述符限制。<br />2.每次要拷贝所有fd。<br />3.每次调用都遍历所有fd集合，耗时长 | 1.适合对实时性要求比较高时。<br />2.需要移植性较强时。 |
| poll   | 遍历     | 链表        | 每次调用都线性遍历所有fd集合。时间复杂度O(n)。               | 无限制     | 每次调用，都需要把fd集合从用户态拷贝到内核态                 | 相比epoll没有最大描述符限制。                                | 1.每次要拷贝所有fd。<br />2.每次调用都遍历所有fd集合，耗时长 | 若仅支持poll和epoll，且对实时性要求不高。              |
| epoll  | 回调     | 红黑树+链表 | 事件驱动，当fd就绪，系统注册的回调函数就会被调用，将就绪fd放到链表中，时间复杂度O(1)。 | 无限制     | 调用epoll_ctl时拷贝fd进内核并保存。调用epoll_wait时不再拷贝。 | 1.最大连接数无限制。<br />2.每个fd只需要拷贝一次<br />3.使用回调，使得无需遍历所有fd集合 | 1. 监听fd数小于1000个时不能体现出优势。<br />2.当监听的描述符状态变化多且描述符存在时间短暂时，也没必要使用epoll。因为此时会进行多次epoll_ctl()系统调用，效率会降低。 | 描述符多、连接大部分是长连接、状态变化较少时。         |
| kqueue | 回调     | 哈希表+链表 | 事件驱动，当fd就绪，系统注册的回调函数就会被调用，将就绪fd放到链表中，时间复杂度O(1) | 无限制     | 调用kqueue时将changelist中的fd拷贝进内核                     | -                                                            | -                                                            | -                                                      |



Refs：

- http://www.10tiao.com/html/548/201710/2651969374/1.html
- https://zhuanlan.zhihu.com/p/22834126
- https://github.com/yuyilei/Daily-Notes/blob/master/md/IO-Multiplexing.md
- https://blog.csdn.net/tianjing0805/article/details/76021440
- https://cyc2018.github.io/CS-Notes/#/notes/Socket?id=%E5%BA%94%E7%94%A8%E5%9C%BA%E6%99%AF

