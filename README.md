Libco
===========
Libco is a c/c++ coroutine library that is widely used in WeChat services. It has been running on tens of thousands of machines since 2013.

By linking with libco, you can easily transform synchronous back-end service into coroutine service. The coroutine service will provide out-standing concurrency compare to multi-thread approach. With the system hook, You can easily coding in synchronous way but asynchronous executed.

You can also use co_create/co_resume/co_yield interfaces to create asynchronous back-end service. These interface will give you more control of coroutines.

By libco copy-stack mode, you can easily build a back-end service support tens of millions of tcp connection.
***
### 简介
libco是微信后台大规模使用的c/c++协程库，2013年至今稳定运行在微信后台的数万台机器上。  

libco通过仅有的几个函数接口 co_create/co_resume/co_yield 再配合 co_poll，可以支持同步或者异步的写法，如线程库一样轻松。同时库里面提供了socket族函数的hook，使得后台逻辑服务几乎不用修改逻辑代码就可以完成异步化改造。

作者: sunnyxu(sunnyxu@tencent.com), leiffyli(leiffyli@tencent.com), dengoswei@gmail.com(dengoswei@tencent.com), sarlmolchen(sarlmolchen@tencent.com)

PS: **近期将开源PaxosStore，敬请期待。**

### libco的特性
- 无需侵入业务逻辑，把多进程、多线程服务改造成协程服务，并发能力得到百倍提升;
- 支持CGI框架，轻松构建web服务(New);
- 支持gethostbyname、mysqlclient、ssl等常用第三库(New);
- 可选的共享栈模式，单机轻松接入千万连接(New);
- 完善简洁的协程编程接口
 * 类pthread接口设计，通过co_create、co_resume等简单清晰接口即可完成协程的创建与恢复；
 * __thread的协程私有变量、协程间通信的协程信号量co_signal (New);
 * 语言级别的lambda实现，结合协程原地编写并执行后台异步任务 (New);
 * 基于epoll/kqueue实现的小而轻的网络框架，基于时间轮盘实现的高性能定时器;

### Build

```bash
$ cd /path/to/libco
$ make
```

or use cmake

```bash
$ cd /path/to/libco
$ mkdir build
$ cd build
$ cmake ..
$ make
```


# notes
## 阻塞与非阻塞
同步调用read使用同步阻塞IO，kernel等待数据到达，再将数据拷贝到用户线程，这两个阶段用户线程都被阻塞。异步调用read使用异步IO模型，用户线程调用read后，立刻可以去做其它事， kernel等待数据准备完成，然后将数据拷贝到用户内存，都完成后，kernel通知用户read完成，然后用户线程直接使用数据，两个阶段用户线程都不被阻塞。而协程调用read使用多路复用IO模型，用户线程调用read后，第一阶段也不会被阻塞，但第二个阶段会被阻塞，epoll多路复用IO模型可以在一个线程管理多个socket。

同步调用在两个阶段都会阻塞用户线程，因此效率低。虽然可以为每个连接开个线程，但连接数多时，线程太多导致性能压力，也可以开固定数目的线程池，但如果存在大量长连接，线程资源不被释放，后续的连接得不到处理。异步调用时，因为两个阶段都不阻塞用户线程，因此效率最高，但异步的调用逻辑和回调逻辑需要分开，在异步调用多时，代码结构不清晰。而协程的epoll多路复用IO模型，虽然会阻塞第二个阶段，但因为第二阶段读数据耗时很少，因此效率略低于异步调用。协程最大的优点是在接近异步效率的同时，可以使用同步的写法（仅仅是同步的写法，不是同步调用）。例如read函数的调用代码后，紧接着可以写处理数据的逻辑，不用再定义回调函数。调用read后协程挂起，其他协程被调用，数据就绪后在read后面处理数据。

## IO多路复用
系统select/poll、epoll函数都提供多路IO复用方案，协程使用的是epoll。select、poll类似，监视writefds、readfds、exceptfds文件描述符(fd)。调用select/poll会阻塞，直到有fd就绪（可读、可写、有except），或超时。select/poll返回后，通过遍历fd集合，找到就绪的fs，并开始读写。poll用链表存储fd，而select用fd标记位存储，因此poll没有最大连接数限制，而select限制为1024。select/poll共有的缺点是：一，返回后需要遍历fd集合找到就绪的fd，但fd集合就绪的描述符很少；二，select/poll均需将fd集合在用户态和内核态之间来回拷贝。epoll的引入是为了解决select/poll上述两个缺点，epoll提供三个函数epoll_create、epoll_ctl、epoll_wait。epoll_create在内核的高速cache中建一棵红黑树以及就绪链表(activeList)。epoll_ctl的add在红黑树上添加fd结点，并且注册fd的回调函数，内核在检测到某fd就绪时会调用回调函数将fd添加到activeList。epoll_wait将activeList中的fd返回。epoll_ctl每次只往内核添加红黑树节点，不需像select/poll拷贝所有fd到内核，epoll_wait通过共享内存从内核传递就绪fd到用户，不需像select/poll拷贝出所有fd并遍历所有fd找到就绪的。





