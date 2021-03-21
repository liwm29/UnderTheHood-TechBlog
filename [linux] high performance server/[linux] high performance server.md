# 高性能linux服务器





## 服务器监听范式

一个传统的单线程服务器

```mermaid
graph LR;
A["socket()"]-->B(sockfd);
B-->|"setsockopt()"| C[bind]
C-->D[listen]
D-->E[accept]
E-->|connfd|F["dowork(){read/write connfd}"]
F-->|"while (1)"| E
```

一个传统的多线程服务器, pthread也可以换成fork,多进程

```mermaid
graph LR;
A["socket()"]-->B(sockfd);
B-->|"setsockopt()"| C[bind]
C-->D[listen]
D-->E[accept]
E-->|connfd|F[pthread_create]
F-->|pthread|G[threadRoutine]-->O["dowork(){read/write connfd}"]
F-->|pthread|H[threadRoutine]-->Oo["dowork(){read/write connfd}"]
F-->|pthread|I[threadRoutine]-->Ooo["dowork(){read/write connfd}"]
F-->|"while (1)"| E
```

一个传统的多线程服务器,多线程同时accpet同一个sockfd

> 这种方法应该会触发所谓的__"惊群现象"__,在linux2.6后,如果是多进程调用sockfd.accept(),则惊群被解决
>
> 如果是使用epoll_wait()监听sockfd,则仍然存在惊群问题
>
> 解决方法是对多个线程,任何时刻都只让一个线程去epoll_wait sockfd

```mermaid
graph LR;
A["socket()"]-->B(sockfd);
B-->|"setsockopt()"| C[bind]
C-->D[listen]-->F[pthread_create]
F-->|pthread|G[threadRoutine]-->aa[accept]-->|connfd|O["dowork(){read/write connfd}"]-->|"while (1)"|aa
F-->|pthread|H[threadRoutine]-->aaa[accept]-->|connfd|Oo["dowork(){read/write connfd}"]-->|"while (1)"|aaa
F-->|pthread|I[threadRoutine]-->aaaa[accept]-->|connfd|Ooo["dowork(){read/write connfd}"]-->|"while (1)"|aaaa
```

## SO_REUSEPORT&SO_REUSEADDR

`SO_REUSEADDR` 常见的是用于复用监听TIME_WAIT状态的端口,对于多线程监听同一个端口,也需要使用这个参数

`SO_REUSEPORT`

为了解决上述的常见的多线程处理网络请求的需求,linux推出了一个sockopt参数: 

首先给出官方文档的介绍,很清晰了

> SO_REUSEPORT (since Linux 3.9)
>
> Permits multiple AF_INET or AF_INET6 sockets to be bound
>               to an identical socket address.  This option must be set
>               on each socket (including the first socket) prior to
>               calling bind(2) on the socket.  To prevent port hijacking,
>               all of the processes binding to the same address must have
>               the same effective UID.  This option can be employed with
>               both TCP and UDP sockets.
>
> For TCP sockets, this option allows accept(2) load
>           distribution in a multi-threaded server to be improved by
>           using a distinct listener socket for each thread.  This
>           provides improved load distribution as compared to
>           traditional techniques such using a single accept(2)ing
>           thread that distributes connections, or having multiple
>           threads that compete to accept(2) from the same socket.
>
> For UDP sockets, the use of this option can provide better
>       distribution of incoming datagrams to multiple processes
>       (or threads) as compared to the traditional technique of
>       having multiple processes compete to receive datagrams on
>       the same socket.

在go中如何实现呢?我们知道一般提供的都是linux c的api,对于go,当然可以通过syscall,但还是会有些不同

具体可以看看这个repo: https://github.com/kavu/go_reuseport/blob/47bb7f1bfa3921a92422a1eb4f0941e9caed1103/tcp.go#L96

如何完成syscall得到的fd与go语言内置的net.Listener之间的转换,可能是一个关键

在该库中,是这样实现

> 由于SO_REUSEPORT在go中尚未提供,所有直接用 var reusePort = 0x0F 代替

```mermaid
graph TD;
fx["begin"]-->|"syscall.ForkLock.RLock()"|a
a["syscall.Socket(soType, syscall.SOCK_STREAM, syscall.IPPROTO_TCP)"]-->|"syscall.ForkLock.RUnlock()"|b[fd]
b-->c["syscall.SetsockoptInt(fd, syscall.SOL_SOCKET, syscall.SO_REUSEADDR, 1)"]
c-->d["syscall.SetsockoptInt(fd, syscall.SOL_SOCKET, reusePort, 1)"]
d-->f["syscall.Bind(fd, sockaddr)"]
f-->g["syscall.Listen(fd, listenerBacklogMaxSize)"]
g-->s["file = os.NewFile(uintptr(fd), getSocketFileName(proto, addr))"]
s-->ss["listener, err = net.FileListener(file)"]
ss-->sss["net.Listener"]

```

这里

- `getSocketFileName()`
  - 只是单纯的返回了`fmt.Sprintf("reuseport.%d.%s.%s", os.Getpid(), proto, addr)`
- `os.NewFile(fd uintptr, name string)*os.File`
  - returns a new File with the given file descriptor and name.`	

- `net.FileListener(* os.File)`
  - `FileListener returns a copy of the network listener corresponding to the open file f`

- `listenerBacklogMaxSize`
  - `fd, err := os.Open("/proc/sys/net/core/somaxconn") 默认最大128`

