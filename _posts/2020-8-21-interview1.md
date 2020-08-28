---
layout: post
title: 【面试】面经总结（一）
tags: interview
stickie: true
---

### 1. LRU 算法

`LRU(Least Recently Used)`缓存算法，最近最久未使用。常用于页面置换算法。是为虚拟页式存储管理服务的。对于在内存中又不用的数据块，称为`LRU`，操作系统会根据哪些数据属于`LRU`而将其移出内存。实现的基本方法有以下几种。

#### a.单链表实现

##### PUT 操作

1. 如果要`put`的元素已经在链表中，需要在原来的位置把旧的数据删除，然后在链表的头部插入新的数据。
2. 如果要`put`的元素不在链表中，则需要先判断缓存区是否已满，如果满的话，则把链表尾部的节点删除，之后把新的数据插入到链表头部。如果没有满的话，则直接把数据插入链表头部即可。

##### GET 操作

1. 如果要`get(key)` 的数据存在于链表中，则把 `value` 返回，并且把该节点删除，删除之后把它插入到链表的头部。
2. 如果要`get(key)`的数据不存在于链表中，则直接返回`-1`即可。

>时间复杂度为`O(n)`，空间复杂度为`O(1)`

#### b.空间换时间

在实际的应用中，`put` 操作一般伴随着` get` 操作，也就是说，`get` 操作的次数是比较多的，而且命中率也是相对比较高的，而 `put` 操作的次数是比较少的，可以考虑采用**空间换时间**的方式来加快`get` 的操作。

要想将`get`操作控制在`O(1)`的时间复杂度，可以使用`Hash`存放`key-value`（Go中的`map`）。用了哈希表之后，虽然能够在` O(1)` 时间内找到目标元素，但是还需要删除对应元素，并且把该元素插入到链表头部，删除一个元素需要定位到这个元素的**前驱**，需要 `O(n)` 时间复杂度的。

#### c.双向链表+哈希表

在删除时，需要额外的指针指在被删除节点的前驱上，此时可以使用**双链表**进行替换。

```go
// LRUCache 包含容量、链表、哈希表
type LRUCache struct {
    capacity int
    evictList *list.List
    items map[int]*list.Element
}
// 内容实体
type Entry struct {
    key string
    value interface{}
}

// Constructor 构造函数
func Constructor(capacity int) LRUCache {
    c := LRUCache{
        capacity: capacity,
        evictList: list.New(),
        items: make(map[int]*list.Element),
    }
    return c
}

// Put 写入新的缓存数据
func (c *LRUCache) Put(key int, value int)  {
    if ent, ok := c.items[key]; ok { 
	      // 如果新写的缓存数据已经存在，则将该数据移至头部
        c.evictList.MoveToFront(ent)
        ent.Value.(*entry).value = value
        return
    }
    
    // 如果不存在，将新元素写入头部
    ent := &entry{key, value}
    entry := c.evictList.PushFront(ent)
    c.items[key] = entry
    
    //  写入后容量超限，则将尾部数据移除
    if c.evictList.Len() > c.capacity {
        c.removeOldest()
    }
}

// Get 获取缓存中的key
func (c *LRUCache) Get(key int) int {
    if ent, ok := c.items[key]; ok { // 如果存在哈希表中
        c.evictList.MoveToFront(ent) // 将该元素移至头部
        return ent.Value.(*entry).value
    }
    return -1
}
// Del 从 cache 中删除 key 对应的元素
func (l *lru) Del(key string) {
	if e, ok := l.cache[key]; ok {
		l.removeElement(e)
	}
}

func (c *LRUCache) removeOldest() {
    ent := c.evictList.Back()
    if ent != nil {
        c.removeElement(ent)
    }
}

func (c *LRUCache) removeElement(e *list.Element) {
    c.evictList.Remove(e)
    kv := e.Value.(*entry)
    delete(c.items, kv.key)
}
```

#### d. LRU 应用场景

1. 底层的内存管理，页面置换算法；
2. 一般的缓存服务，如`memcache`，`redis`等；

### 2. 一致性Hash

#### a. 为什么不能直接使用Hash

直接使用取模`Hash`算法时，会出现一些缺陷，主要体现在服务器数量变动的时候，所有缓存的位置都要发生改变。当服务器数量发生改变时，所有缓存在一定时间内是失效的，当应用无法从缓存中获取数据时，则会向后端数据库请求数据（导致**缓存雪崩**）。

#### b. 一致性Hash

一致性Hash算法也是使用取模的方法，只不过一致性`Hash`算法是对$2^{32}$取模。整个空间按照**顺时针方向组织**。将由这$2^{32}$个点组成的圆环称为**Hash环**。

<img src="https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_jpg/UtWdDgynLdbhiae1AfNYAibdp7ib2wTZTrpb4iaIwTkwrloXHV4ebuemEQsqkhTveAlFCN9TQA4YN5mNbAO1tutyow/640" alt="一致性Hash" style="zoom:50%;" />

使用对应算法定位数据访问到相应服务器：将数据`key`使用相同的函数`Hash`计算出哈希值，并确定此数据在环上的位置，从此位置沿环**顺时针**“行走”，第一台遇到的服务器就是其应该定位到的服务器。

#### c. 一致性Hash的容错性和可扩展性

在一致性Hash算法中，如果一台服务器不可用，则受影响的数据仅仅是此服务器到其环空间中前一台服务器（即沿着逆时针方向行走遇到的第一台服务器）之间数据，其它不会受到影响。

<img src="https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_jpg/UtWdDgynLdbhiae1AfNYAibdp7ib2wTZTrppjapqr7nD1s7BeSGmJKMichyiaQJrIqtatg5WibKY4yYc0bdJjrticu73g/640" alt="一致性Hash的容错性" style="zoom:50%;" />

如果增加一台服务器，则受影响的数据仅仅是新服务器到其环空间中前一台服务器（即沿着逆时针方向行走遇到的第一台服务器）之间数据，其它数据也不会受到影响。

<img src="https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_jpg/UtWdDgynLdbhiae1AfNYAibdp7ib2wTZTrpA5jBRhZxgl6Uh78tMibBKebtnwIQsUibPRV0XExnhTnmgvNbBJSmFicOA/640" alt="一致性Hash扩展性" style="zoom:50%;" />

#### d. Hash 环的数据倾斜问题

一致性Hash算法在**服务节点太少时**，容易因为节点分部不均匀而造成**数据倾斜**（被缓存的对象大部分集中缓存在某一台服务器上）问题。为了解决这种数据倾斜问题，一致性Hash算法引入了**虚拟节点机制**，即对每一个服务节点计算多个哈希，每个计算结果位置都放置一个此服务节点，称为**虚拟节点**。具体做法可以在**服务器IP或主机名的后面增加编号**来实现。

<img src="https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbhiae1AfNYAibdp7ib2wTZTrpnNGzjcWYy3ylL7s1Bq2UKicU5mYG8SHsuIFTOf2PMe2FstpM2gMeQbw/640" alt="解决数据倾斜问题" style="zoom:50%;" />

>在实际应用中，通常将虚拟节点数设置为32甚至更大，因此即使很少的服务节点也能做到相对均匀的数据分布。

#### e.具体实现

```go
type UInt32Slice []uint32

func (s UInt32Slice) Len() int {
    return len(s)
}

func (s UInt32Slice) Less(i, j int) bool {
    return s[i] < s[j]
}

func (s UInt32Slice) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}

type Hash func(data []byte) uint32

type Map struct {
    hash Hash
    replicas int               // 复制因子
    keys UInt32Slice           // 已排序的节点哈希切片
    hashMap map[uint32]string  // 节点哈希和KEY的map，键是哈希值，值是节点Key 
}

func New(replicas int, fn Hash) *Map {
    m := &Map{
        replicas: replicas,
        hash:     fn,
        hashMap:  make(map[uint32]string),
    }
    // 默认使用CRC32算法
    if m.hash == nil {
        m.hash = crc32.ChecksumIEEE
    }
    return m
}

func (m *Map) IsEmpty() bool {
    return len(m.keys) == 0
}
// Add 方法用来添加缓存节点，参数为节点key，比如使用IP
func (m *Map) Add(keys ...string) {
    for _, key := range keys {
        // 结合复制因子计算所有虚拟节点的hash值，并存入m.keys中，同时在m.hashMap中保存哈希值和key的映射
        for i:=0; i<m.replicas; i++ {
            hash := m.hash([]byte(strconv.Itoa(i) + key))
            m.keys = append(m.keys, hash)
            m.hashMap[hash] = key
        }
    }
    // 对所有虚拟节点的哈希值进行排序，方便之后进行二分查找
    sort.Sort(m.keys)
}
// Get 方法根据给定的对象获取最靠近它的那个节点key
func (m *Map) Get(key string) string {
    if m.IsEmpty() {
        return ""
    }
    hash := m.hash([]byte(key))
    // 通过二分查找获取最优节点，第一个节点hash值大于对象hash值的就是最优节点
    // 底层使用的就是 二分法
    // 返回能够使 f(i)=true 的最小的 i（0<=i<n） 值
    idx := sort.Search(len(m.keys), func(i int) bool { return m.keys[i] >= hash })
    // 如果查找结果大于节点哈希数组的最大索引，表示此时该对象哈希值位于最后一个节点之后，那么放入第一个节点中
    if idx == len(m.keys) {
        idx = 0
    }

    return m.hashMap[m.keys[idx]]
}
```

### 3. I/O复用

#### a. Unix中的五种I/O模型

##### 阻塞式I/O

<img src="https://user-gold-cdn.xitu.io/2019/3/31/169d16a9a67b8a0a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1" alt="img" style="display: block; margin: 0 auto; zoom: 67%;" />

具体过程就是首先应用进程发起系统调用，会进入等待，一直等到数据报就绪，这时候数据还是在内核缓冲区中，需要将数据报返回给应用进程缓冲区。

> 阻塞式 I/O 不是意味着系统进入阻塞，而仅仅是当前应用程序阻塞，其他应用程序还是可以继续运行的，因此不消耗 CPU 时间，执行效率较高

##### 非阻塞式I/O

<img src="https://user-gold-cdn.xitu.io/2019/3/31/169d18e8f8033ded?imageView2/0/w/1280/h/960/format/webp/ignore-error/1" alt="img" style="display: block; margin: 0 auto; zoom: 67%;" />

具体过程是采用轮询的方式不断进行系统调用来获取I/O是否完成的结果。

##### I/O复用

<img src="https://user-gold-cdn.xitu.io/2019/3/31/169d19896b6c3b37?imageView2/0/w/1280/h/960/format/webp/ignore-error/1" alt="img" style="display: block; margin: 0 auto; zoom: 67%;" />

具体过程主要是先调用`select`，系统监听所有`select`负责的数据报，一旦有某个数据准备就绪，就会将其返回。然后进行`recvfrom`系统调用，执行同阻塞式`I/O`相同的处理。

> 由于I/O复用需要调用两个系统调用，所以效率肯定不如前者，但是最大的特点就是可以同时处理多个 `connection`。

##### 异步I/O

<img src="https://user-gold-cdn.xitu.io/2019/3/31/169d1a09ec4b4e01?imageView2/0/w/1280/h/960/format/webp/ignore-error/1" alt="img" style="display: block; margin: 0 auto; zoom: 67%;" />

进行 `aio_read` 系统调用会立即返回， 应用进程继续执行， 不会被阻塞， 内核会在所有操作完成之后向应用进程发送信号。

<img src="https://ninokop.github.io/2018/02/18/go-net/io.png" alt="io" style="display: block; margin: 0px auto; zoom: 80%;" />

#### b. 阻塞与非阻塞的区别

阻塞式和非阻塞式最大的区别就是阻塞式在等待数据阶段会进入阻塞，而非阻塞式不会，但是对于非阻塞式，在获得数据存在内核缓冲区后，**将内核缓冲区中数据复制到应用程序缓冲区这个阶段是阻塞的**。

#### c. 同步与异步的区别

同步与异步的区别在于在进行 I/O 操作时候会将进程阻塞，根据这个定义就知道，阻塞式、非阻塞式、信号驱动式、I/O 复用式都属于同步。

#### d.I/O多路复用

##### select

有三种类型的描述符类型： `readset`、 `writeset`、 `exceptset`， 分别对应**读、 写、 异常**条件的描述符集合。 `fd_set` 使用数组实现， 数组大小使用 `FD_SETSIZE` 定义。`timeout` 为超时参数， 调用 `select` 会一直阻塞直到有描述符的事件到达或者等待的时间超过 `timeout`。成功调用返回结果大于 `0`， 出错返回结果为 `-1`， 超时返回结果为 `0`。

```cpp
#include <sys/select.h>

/* According to earlier standards */
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);

// 和 select 紧密结合的四个宏：
void FD_CLR(int fd, fd_set *set);
int FD_ISSET(int fd, fd_set *set);
void FD_SET(int fd, fd_set *set);
void FD_ZERO(fd_set *set);
```

**存在的问题**：

1. 被监控的fds集合限制为1024
2. fds集合需要从用户空间拷贝到内核空间
3. 需要遍历整个fds来收集可读事件

##### poll

`poll`最大的特点在于没有最大文件描述数量的限制。poll改变了fds集合的描述方式，使用了pollfd结构而不是select的fd_set结构，使得poll支持的fds集合限制远大于select的1024，其中 pollfd 使用链表实现。

```cpp
int poll(struct pollfd *fds, unsigned int nfds, int timeout);
```

> select 与 poll 的区别
>
> 1. 功能
>    1. `select` 的描述符类型使用数组实现，`FD_SETSIZE` 默认大小为 `1024`，不过这个值可以改变，如果需要修改的话要重新编译；而 `poll` 使用链表实现，没有描述符大小的限制；
>
> 2. 速度
>    1. 速度都很慢；
>    2. 共同的就是在调用时都需要将全部描述符从应用进程缓冲区复制到内核缓冲区；
>    3. 两者返回结果中没有声明哪些描述符已经准备好，所以如果返回值大于 0 时，应用进程都需要使用轮询的方式来找到 I/O 完成的描述符；

##### epoll

`epoll_ctl()` 用于向内核注册新的描述符或者是改变某个文件描述符的状态。 已注册的描述符在内核中会被维护在一棵红黑树上， 通过回调函数内核会将 I/O 准备好的描述符加入到一个链表中管理， 进程调用 `epoll_wait()` 便可以得到事件完成的描述符。

`epoll` 对多线程编程更有友好， 一个线程调用了 `epoll_wait()` 另一个线程关闭了同一个描述符也不会产生像 `select` 和 `poll` 的不确定情况。

`epoll` 比 `select` 和 `poll` 更加灵活而且没有描述符数量限制。

`epoll` 对多线程编程更有友好， 一个线程调用了 `epoll_wait()` 另一个线程关闭了同一个描述符也不会产生像 `select` 和 `poll` 的不确定情况。

```cpp
#include <sys/epoll.h>
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *evet);
int epoll_wait(int epfd, struct eopll_event *events, int maxevents, int timeout);
```

调用过程：

* 通过`epoll_create()`建立一个`epoll`对象

> `epoll_create()`会创建一个`eventpoll`结构体：
>
> ```c++
> struct eventpoll {
>     ...
>     struct rb_root rbr;
>     struct list_head rdlist;
>     ...
> };
> // 每一个事件都会建立一个 epitem 结构体
> struct epitem {
>     struct rb_node rbn;
>     struct list_head rbllink;
>     struct epoll_filefd ffd;
>     struct eventpoll *ep;
>     struct epoll_event event;
> }
> ```
>
> 所有的事件都会挂载在红黑树中，以便识别出重复添加的事件。

* 调用`epoll_ctl`向`epoll`对象中添加链接的套接字
* 调用`epoll_wait`收集发生的事件的连接

<img src="https://raw.githubusercontent.com/panjf2000/illustrations/master/go/epoll-principle.png?imageView2/2/w/1280/format/jpg/interlace/1/q/100" alt="img" style="display: block; margin: 0px auto; zoom: 80%;" />

##### epoll 工作模式

> 1. LT 模式
>
>    当 `epoll_wait()` 检测到描述符事件到达时，将此时间通知进程，进程**可以**不立即处理该事件，下次调用 `epoll_wait()` 时会再次通知进程，这是默认一种模式，并且同时支持阻塞和非阻塞.
>
> 2. ET 模式
>
>    和 LT 模式不同的是，通知之后必须立即处理事件，下次再调用 `epoll_wait()` 不会再得到时间到达的通知。减少了 `epoll` 事件被重复触发的次数，因此效率比 LT 高，只支持非阻塞式，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。
>

#### e.多路复用应用场景

* `select` 使用场景
  * `select` 的 `timeout` 精度为 1ns，而其他两种为 1ms，所以 `select` 更适用于实时要求很高的场景，比如核反应堆的控制

* `poll` 使用场景
  * `poll` 与 `select` 相比没有最大描述符数量的限制，并且如果平台对实时性要求不是很高，一般使用`poll`
  * 需要同时监控小于 1000 个描述符，就没必要使用 `epoll`，因为这个应用场景下并不能体现 `epoll` 的优势

* `epoll` 使用场景
  * 有非常大量的描述符需要同时轮询，而且这些连接最好是长连接

#### f.服务器常用编程模型

##### 多线程

##### IO多路复用

实际上目前的高性能服务器很多都用的是`reactor`模式，即`non-blocking IO`+`IO multiplexing`的方式。通常主线程只做`event-loop`，通过`epoll_wait`等方式监听事件，而处理客户请求是在其他工作线程中完成。

> 传统的网络实现
>
> 1. `server`端在`bind&listen`后，将`listenfd`注册到`epollfd`中，最后进入`epoll_wait`循环。循环过程中若有在`listenfd`上的`event`则调用`socket.accept`，并将`connfd`加入到`epollfd`的IO复用队列。
> 2. 当`connfd`上有数据到来或写缓冲有数据可以触发`epoll_wait`的返回，这里读写IO都是非阻塞IO，这样才不会阻塞`epoll`的下一个循环。然而，这样容易割裂业务逻辑，不易理解和维护。
> 3. `read`后数据进行解码并放入队列中，等待工作线程处理。

<img src="https://ninokop.github.io/2018/02/18/go-net/c-net.jpg" alt="io" style="display: block; margin: 0px auto;" />

#### g. Go 中的 netPoll

```go
func main() {
    ln, err := net.Listen("tcp", ":8080")
    for {
        conn, _ := ln.Accept()
        go echoFunc(conn)
    }
}
```

总结来说，所有的网络操作都以网络描述符`netFD`为中心实现。`netFD`与底层`PollDesc`结构绑定，当在一个`netFD`上读写遇到`EAGAIN`错误时，就将当前`goroutine`存储到这个`netFD`对应的`PollDesc`中，同时将`goroutine`给`park`住，直到这个`netFD`上再次发生读写事件，才将此`goroutine`给`ready`激活重新运行。显然，在底层通知`goroutine`再次发生读写等事件的方式就是`epoll`等事件驱动机制。

##### netFD

在Go中，服务端通过`listen()`建立起的`Listener`是个实现了`Accept()` `Close()`等方法的接口。通过`Listener`的`Accept()`方法返回的`Conn`是一个实现了`Read()` `Write()`等方法的接口。`Listener`和`Conn`的实现都包含一个网络文件描述符`netFD`，它其中包含一个重要的数据结构`pollDesc`，它是底层事件驱动的封装。

```go
type TCPListener struct {
    fd *netFD
    lc ListenConfig
}
// Network file descriptor.
type netFD struct {
	pfd poll.FD
	// immutable until Close
	family      int
	sotype      int
	isConnected bool // handshake completed or use of association with peer
	net         string
	laddr       Addr
	raddr       Addr
}

// FD is a file descriptor. The net and os packages use this type as a
// field of a larger type representing a network connection or OS file.
type FD struct {
	// Lock sysfd and serialize access to Read and Write methods.
	fdmu fdMutex
	// System file descriptor. Immutable until Close.
	Sysfd int
	// I/O poller.
	pd pollDesc
	// Writev cache.
	iovecs *[]syscall.Iovec
	// Semaphore signaled when file is closed.
	csema uint32
	// Non-zero if this file has been set to blocking mode.
	isBlocking uint32
	// Whether this is a streaming descriptor, as opposed to a
	// packet-based descriptor like a UDP socket. Immutable.
	IsStream bool
	// Whether a zero byte read indicates EOF. This is false for a
	// message based socket connection.
	ZeroReadIsEOF bool
	// Whether this is a file rather than a network socket.
	isFile bool
}
```

1. 服务端的`netFD`在`listen`时会创建`epoll`的实例，并将`listenFD`加入`epoll`的事件队列；
2. `netFD`在`accept`时将返回的`connFD`也加入`epoll`的事件队列；
3. `netFD`在读写时出现`syscall.EAGAIN`错误，通过`pollDesc`将当前的`goroutine` `park`住，直到`ready`，从`pollDesc`的`waitRead`中返回；

##### pollDesc

```go
// Network poller descriptor.
//
// No heap pointers.
//
//go:notinheap
type pollDesc struct {
	link *pollDesc // in pollcache, protected by pollcache.lock

	// The lock protects pollOpen, pollSetDeadline, pollUnblock and deadlineimpl operations.
	// This fully covers seq, rt and wt variables. fd is constant throughout the PollDesc lifetime.
	// pollReset, pollWait, pollWaitCanceled and runtime·netpollready (IO readiness notification)
	// proceed w/o taking the lock. So closing, everr, rg, rd, wg and wd are manipulated
	// in a lock-free way by all operations.
	// NOTE(dvyukov): the following code uses uintptr to store *g (rg/wg),
	// that will blow up when GC starts moving objects.
	lock    mutex // protects the following fields
	fd      uintptr
	closing bool
	everr   bool    // marks event scanning error happened
	user    uint32  // user settable cookie
	rseq    uintptr // protects from stale read timers
	rg      uintptr // pdReady, pdWait, G waiting for read or nil
	rt      timer   // read deadline timer (set if rt.f != nil)
	rd      int64   // read deadline
	wseq    uintptr // protects from stale write timers
	wg      uintptr // pdReady, pdWait, G waiting for write or nil
	wt      timer   // write deadline timer
	wd      int64   // write deadline
}
```

`runtime.pollDesc` 包含自身类型的一个指针，用来保存下一个 `runtime.pollDesc` 的地址，以此来实现链表，可以减少数据结构的大小，所有的 `runtime.pollDesc` 保存在 `runtime.pollCache` 结构中，定义如下：

```go
type pollCache struct {
	lock  mutex
	first *pollDesc
	// PollDesc objects must be type-stable,
	// because we can get ready notification from epoll/kqueue
	// after the descriptor is closed/reused.
	// Stale notifications are detected using seq variable,
	// seq is incremented when deadlines are changed or descriptor is reused.
}
```

##### netpoll_epoll

在`linux`上是通过`runtime`包中的`netpoll_epoll.go`实现。它封装了`epoll`的三个系统调用，即`epoll_create` `epoll_ctl`和`epoll_wait`。

```cpp
#include <sys/epoll.h>  
int epoll_create(int size);  
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);  
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);

// Go 对上面三个调用的封装
func netpollinit()
func netpollopen(fd uintptr, pd *pollDesc) int32
func netpoll(block bool) gList
```

它封装成了四个`runtime`函数。`netpollinit()` 使用`epoll_create`创建`epollfd`，`netpollopen()` 添加一个`fd`到`epoll`中，这里的数据结构称为`pollDesc`，它一开始都关注了**读写事件**，并且采用的是**边缘触发**。`netpollclose()`函数就是从`epoll`删除一个`fd`。`netpoll()` 就是从`epoll wait`得到所有发生事件的`fd`，并将每个`fd`对应的`goroutine`通过链表返回。

### 4. 线程池（协程池）

#### a. 什么是线程池

线程池是一种多线程处理形式，处理过程中将任务添加到队列，然后在创建线程后自动启动这些任务。

#### b. 为什么使用线程池

* 降低资源消耗
  * 通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
* 提高响应速度
  * 当任务到达时，任务可以不需要等到线程创建就能立即执行。
* 提高线程的可管理性
  * 线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。

#### b. 最简单的方法

思路其实非常简单，用一个`channel`当做任务队列，初始化`groutine`池时确定好并发量，然后以设置好的并发量开启`groutine`同时读取`channel`中的任务并执行。

```go
type SimplePool struct {
    // 控制并发量
    wg sync.WaitGroup
    // 任务队列
    work chan func()
}

func New(workers int) *SimplePool {
    p := &SimplePool{
        wg: sync.WaitGroup{},
        work: make(chan func()),
    }
    p.wg.Add(workers)
    //根据指定的并发量去读取管道并执行
    for i := 0; i < workers; i++ {
        go func() {
            defer func() {
                // 捕获异常 防止waitGroup阻塞
                if err := recover(); err != nil {
                    fmt.Println(err)
                    p.wg.Done()
                }
            }()
            // 从workChannel中取出任务执行
            for fn := range p.work {
                fn()
            }
            p.wg.Done()
        }()
    }
    return p
}

// 添加任务
func (p *SimplePool) Add(fn func()) {
    p.work <- fn
}

// 执行
func (p *SimplePool) Run() {
    close(p.work)
    p.wg.Wait()
}
```

#### b. 进阶版

第一种方法虽然简单，但是对每一个并发任务的状态，`pool`的状态缺少控制。因此可以考虑添加并发安全的原子操作值来标识其运行状态。

```go
// 需要加入pool 中执行的任务
type WorkFunc func(wu WorkUnit) (interface{}, error)

// 工作单元
type workUnit struct {
    value      interface{}    // 任务结果 
    err        error          // 任务的报错
    done       chan struct{}  // 通知任务完成
    fn         WorkFunc    
    cancelled  atomic.Value   // 任务是否被取消
    cancelling atomic.Value   // 是否正在取消任务
    writing    atomic.Value   // 任务是否正在执行
}
// 线程池结构
type limitedPool struct {
    workers uint            // 并发量 
    work    chan *workUnit  // 任务channel
    cancel  chan struct{}   // 用于通知结束的channel
    closed  bool            // 是否关闭
    m       sync.RWMutex    // 读写锁，主要用来保证 closed值的并发安全
}
```

初始化`goroutine`池，以及启动设定好数量的`goroutine`。

```go
// 初始化pool,设定并发量
func NewLimited(workers uint) Pool {
    if workers == 0 {
        panic("invalid workers '0'")
    }
    p := &limitedPool{
        workers: workers,
    }
    p.initialize()
    return p
}

func (p *limitedPool) initialize() {
    p.work = make(chan *workUnit, p.workers*2)
    p.cancel = make(chan struct{})
    p.closed = false
    for i := 0; i < int(p.workers); i++ {
        // 初始化并发单元
        p.newWorker(p.work, p.cancel)
    }
}

// passing work and cancel channels to newWorker() to avoid any potential race condition
// betweeen p.work read & write
func (p *limitedPool) newWorker(work chan *workUnit, cancel chan struct{}) {
    go func(p *limitedPool)  {
        var wu *workUnit
        defer func(p *limitedPool) {
            if err := recover(); err != nil {
                trace := make([]byte, 1<<16)
                n := runtime.Stack(trace, true)
                s := fmt.Sprintf(errRecovery, err, string(trace[:int(math.Min(float64(n), float64(7000)))]))

                iwu := wu
                iwu.err = &ErrRecovery{s: s}
                close(iwu.done)

                // need to fire up new worker to replace this one as this one is exiting
                p.newWorker(p.work, p.cancel)
            }
        }(p)
        var value interface{}
        var err error
        for {
            select {
            // workChannel中读取任务
            case wu = <-work:
                // 防止channel 被关闭后读取到零值
                if wu == nil {
                    continue
                }
                // 先判断任务是否被取消
                if wu.cancelled.Load() == nil {
                    // 执行任务
                    value, err = wu.fn(wu)
                    wu.writing.Store(struct{}{})                    
                    // 任务执行完在写入结果时需要再次检查工作单元是否被取消，防止产生竞争条件
                    if wu.cancelled.Load() == nil && wu.cancelling.Load() == nil {
                        wu.value, wu.err = value, err
                        close(wu.done)
                    }
                }
            // pool是否被停止
            case <-cancel:
                return
            }
        }
    }(p)
}
```

往`pool`中添加任务，并检查`pool`是否已经关闭。

```go
func (p *limitedPool) Queue(fn WorkFunc) WorkUnit {
    w := &workUnit{
        done: make(chan struct{}),
        fn:   fn,
    }
    go func() {
        p.m.RLock()
        if p.closed {
            w.err = &ErrPoolClosed{s: errClosed}
            if w.cancelled.Load() == nil {
                close(w.done)
            }
            p.m.RUnlock()
            return
        }
        // 将工作单元写入workChannel, pool启动后将由上面newWorker函数中读取执行
        p.work <- w
        p.m.RUnlock()
    }()
    return w
}
```

### 5. HTTP 与 HTTPS

<img src="https://upload-images.jianshu.io/upload_images/16796915-df1c6921f2c14805.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="io" style="display: block; margin: 0px auto;" />

#### a. HTTPS 解决的问题

* 信息加密传输：第三方无法窃听；
* 校验机制：一旦被篡改，通信双方会立刻发现；
* 身份证书：防止身份被冒充。

#### b. HTTPS 过程

1. 客户端请求服务器获取[证书公钥]()
2. 客户端(`SSL`/`TLS`)解析证书（无效会弹出警告）
3. 生成随机值
4. 用[公钥加密]()随机值生成**密钥**
5. 客户端将[秘钥]()发送给服务器
6. 服务端用[私钥]()解密[秘钥]()得到随机值
7. [将信息和随机值混合在一起]()进行对称加密
8. 将加密的内容发送给客户端
9. 客户端用[秘钥]()解密信息

### 6.数据库范式

#### a. 第一范式

每一列属性都是不可再分的属性值，确保每一列的原子性。

#### b. 第二范式

在满足第一范式的基础上，实体的每个非主键属性完全函数依赖于主键属性（消除部分依赖）。

>第二范式产生的影响
>
>* 数据信息冗余
>* 增删改会出现问题

解决办法：使用关系分解方法消除部分依赖

#### c. 第三范式

在满足第二范式的基础上，在实体中不存在非主键属性传递函数依赖于主键属性。（表中字段[非主键]()不存在对主键的传递依赖）

> 第三范式产生的影响
>
> * 更新时需要同时修改多表记录，否则会出现数据不一致的情况

解决办法：使用关系分解方法消除部分依赖

#### d. 什么时候需要突破范式

在概念数据模型设计时遵守第三范式，降低范式标准的工作放到物理数据模型设计时考虑。降低范式就是增加字段，允许冗余，达到以空间换时间的目的。

#### e.范式化与反范式化设计的优缺点

范式化设计的优点在于：

* 可以尽量的减少数据冗余，数据表更新快体积小
* 范式化的更新操作比反范式化更快

缺点在于：

* 对于查询需要对多个表进行关联
* 索引优化较难

反范式化优点在于：

* 可以减少表的关联
* 可以更好的进行索引优化

缺点在于：

* 存在数据冗余及数据维护的异常
* 对数据的修改需要更多的成本

### 7. 索引创建与优化

#### a. 分析原因

1. 先观察，开启慢查询日志，设置相应的阈值（比如超过3秒就是慢`SQL`），在生产环境跑上个一天过后，看看哪些`SQL`比较慢

   > 慢查询的相关参数：
   >
   > 1. `slow_query_log`：是否开启慢查询日志
   > 2. `slow_query_log_file`：MySQL数据库慢查询日志存储路径，存在默认值
   > 3. `slow_query_time`：慢查询阈值
   > 4. `log_queries_not_using_indexes`：未使用索引的查询也被记录到慢查询日志中
   > 5. `log_output`：日志存储方式，包括`FILE`、`TABLE`等

2. `Explain`和慢`SQL`分析

3. `Show Profile`是比`Explain`更近一步的执行细节，可以查询到执行每一个`SQL`都干了什么事，这些事分别花了多少秒

#### b. Explain 详解

* id
  * `id` 相同，执行顺序由上而下
  * `id` 不同，值越大越先被执行
* select_type
  * 可以查看 `id` 的执行实例
* table
  * `table`表示查询所涉及的表或衍生的表
* type
  * 通过`type`字段，可以判断此次查询是**全表扫描**还是**索引扫描**
  * `system`：表中只有一条数据，这个类型是特殊的`const`的类型。
  * `const`：针对主键或唯一索引的等值查询扫描，最多只返回一行数据。
  * `eq_ref`：此类型通常出现在多表的 join 查询，表示对于前表的每一个结果，都只能匹配到后表的一行结果。
  * `ref`：此类型通常出现在多表的 join 查询，针对于非唯一或非主键索引。
  * `range`：表示使用索引范围查询，通过索引字段范围获取表中部分数据记录。
  * `index`：表示全索引扫描(`full index scan`)，和 `ALL` 类型类似，只不过 `ALL` 类型是全表扫描，而 `index` 类型则仅仅扫描所有的索引， 而不扫描数据。
  * `ALL`：表示全表扫描，这个类型的查询是性能最差的查询之一。

> 通常来说, 不同的 type 类型的性能关系如下:
>
> ALL < index < range ~ index_merge < ref < eq_ref < const < system

* possible_keys
  * 表示在查询时，可能使用到的索引。

* key
  * 此字段是在当前查询时所真正使用到的索引。
* key_len
  * 表示查询优化器使用了索引的字节数，这个字段可以评估组合索引是否完全被使用。
* ref
  * 表示显式索引的哪一列被使用了。
* rows
  * 用于估算 `sql` 要查找到结果集需要扫描读取的数据行数。

* extra
  * `using filesort`：表示需要额外的排序操作，不能通过索引顺序达到排序效果。
  * `using index`：覆盖索引扫描，表示查询在索引树中就可查找所需数据。
  * `using temporary`：查询有使用临时表。
  * `using where`：表名使用了where过滤。

#### c. 是否需要创建索引

| 需要创建索引                                                 | 不需要创建索引                                               |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 主键自动建立唯一索引                                         | 表记录过少                                                   |
| 频繁作为查询条件的字段应该创建索引                           | 经常增删改的表或者字段（虽然可以提高查询速度，但是会降低更新表的速度，因为更新表时，不仅要保存数据，还要保存一下索引文件） |
| 查询中与其它表关联的字段，外键关系建立索引                   | `where`条件里用不到的字段不创建索引                          |
| 查询中排序的字段，排序字段若通过索引去访问将大大提高排序速度 | 过滤性不好的不适合建索引，如性别等                           |

#### d. SQL语法顺序&执行顺序

```sql
# 语法顺序
SELECT
DISTINCT <select_list>
FROM <left_table> <join_type> JOIN <right_table>
ON <join_condition>
WHERE <where_condition>
GROUP BY <groub_by_list>
HAVING <having_condition>
ORDER BY <order_by_condition>
LIMIT <limit_number>

# 执行顺序
FROM <表名> # 选取表，将多个表数据通过笛卡尔积变成一个表。 
ON <筛选条件> # 对笛卡尔积的虚表进行筛选 
JOIN <join, left join, right join...> <join表> # 指定join，用于添加数据到on之后的虚表中，例如left join会将左表的剩余数据添加到虚表中 
WHERE <where条件> # 对上述虚表进行筛选 
GROUP BY <分组条件> # 分组 
<SUM()等聚合函数> # 用于having子句进行判断，在书写上这类聚合函数是写在having判断里面的 
HAVING <分组筛选> # 对分组后的结果进行聚合筛选 
SELECT <返回数据列表> # 返回的单列必须在group by子句中，聚合函数除外 
DISTINCT # 数据除重 
ORDER BY <排序条件> # 排序 
LIMIT <行数限制> 
```

#### e. 避免不走索引的场景

* 尽量避免在字段开头模糊查询，会导致数据库引擎放弃索引进行全表扫描。

```SQL
SELECT * FROM t WHERE username LIKE '%陈%';
SELECT * FROM t WHERE username LIKE '陈%';
```

如果要求使用模糊查询，则使用`MySQL`内置函数`INSTR(str, substr)`来匹配。数据量较少时，直接用`like '%xx%'`即可。

* 尽量避免使用 `in` 和 `not in`，会导致引擎走全表扫描

```sql
SELECT * FROM t WHERE id IN (2,3);
SELECT * FROM t WHERE id BETWEEN 2 AND 3;
```

子查询可以使用`exists`代替。

```sql
-- 不走索引 
select * from A where A.id in (select id from B); 
-- 走索引 
select * from A where exists (select * from B where B.id = A.id);
```

* 尽量避免使用 `or`，会导致数据库引擎放弃索引进行全表扫描

```sql
SELECT * FROM t WHERE id = 1 OR id = 3 
SELECT * FROM t WHERE id = 1 
   UNION 
SELECT * FROM t WHERE id = 3 
```

* 尽量避免进行 null 值的判断，会导致数据库引擎放弃索引进行全表扫描

```sql
SELECT * FROM t WHERE score IS NULL
SELECT * FROM t WHERE score = 0
```

* 尽量避免在 where 条件中等号的左侧进行表达式、函数操作，会导致数据库引擎放弃索引进行全表扫描

```sql
-- 全表扫描 
SELECT * FROM T WHERE score/10 = 9 
-- 走索引 
SELECT * FROM T WHERE score = 10*9 
```

* 当数据量大时，避免使用 `where 1=1` 的条件

> 什么是`where 1=1`
>
> ☆在查询条件数量不确定的条件情况下，使用 where 1=1可以很方便的规范语句。

* 查询条件不能用 <> 或者 !=

* `where` 条件仅包含复合索引非前置列

> 什么是最左匹配原则
>
> 1. 如果创建一个联合索引, 那这个索引的任何前缀都会用于查询，`(col1,col2, col3)`这个联合索引的所有前缀 就是`(col1), (col1, col2), (col1, col2, col3)`， 包含这些列的查询都会启用索引查询。
> 2. 其他所有不在最左前缀里的列都不会启用索引, 即使包含了联合索引里的部分列也不行。
>
> 还需要注意的：
>
> 1. 联合索引最多只能包含16列；
> 2. `blob`和`text`也能创建索引, 但是必须指定前面多少位；

* 隐式类型转换造成不使用索引
* `order by` 条件要与 `where` 中条件一致，否则 `order by` 不会利用索引进行排序

```sql
-- 不走age索引 
SELECT * FROM t order by age; 
-- 走age索引 
SELECT * FROM t where age > 0 order by age; 
```

当 `order by` 中的字段出现在` where` 条件中时，才会利用索引而不再二次排序，更准确的说，`order by` 中的字段在执行计划中利用了索引时，不用排序操作。

#### f. SELECT 语句优化

* 避免出现 `select *`

使用 `select *` 取出全部列，会让优化器无法完成索引覆盖扫描这类优化，会影响优化器对执行计划的选择，也会增加网络带宽消耗，更会带来额外的 I/O，内存和 CPU 消耗。

* 避免出现不确定结果的函数

特定针对主从复制这类业务场景。由于原理上从库复制的是主库执行的语句，使用如 `now()`、`rand()`、`sysdate()`、`current_user()` 等不确定结果的函数很容易导致主库与从库相应的数据不一致。

* 多表关联查询时，小表在前，大表在后

在 `MySQL` 中，执行 `from` 后的表关联查询是从左往右执行的，第一张表会涉及到全表扫描。

* 使用表的别名
* 用 where 字句替换 HAVING 字句

> `where`与`having`的区别
>
> 本质的区别就是where筛选的是数据库表里面本来就有的字段，而having筛选的字段是从前筛选的字段筛选的。
>
> ```sql
> -- 既可以使用 where 也可以使用 having 的情况
> select goods_price,goods_name from sw_goods where goods_price>100
> select goods_price,goods_name from sw_goods having goods_price>100
> -- 只可以使用 where 的情况
> select goods_name,goods_number from sw_goods where goods_price>100
> select goods_name,goods_number from sw_goods having goods_price>100(X)
> ```

* 调整 `where` 字句中的连接顺序

`MySQL` 采用从左往右，自上而下的顺序解析 `where` 子句。根据这个原理，应将过滤数据多的条件往前放，最快速度缩小结果集。

#### g. 增删改DML语句优化

* 大批量插入数据

如果同时执行大量的插入，建议使用多个值的 INSERT 语句。

```sql
INSERT INTO t VALUES(1,2);
INSERT INTO t VALUES(1,3);
INSERT INTO t VALUES(1,4);

INSERT INTO t VALUES(1,2),(1,3),(1,4);  
```

* 避免重复查询更新的数据

#### h.查询条件优化

* 对于复杂的查询，可以使用中间临时表暂存数据
* 优化 group by 语句

默认情况下，MySQL 会对 `GROUP BY` 分组的所有值进行排序。如果查询包括 `GROUP BY` 但你并不想对分组的值进行排序，你可以指定` ORDER BY NULL` 禁止排序。

```sql
SELECT col1, col2, COUNT(*) FROM table GROUP BY col1, col2 ORDER BY NULL; 
```

* 优化 `join` 语句

使用`join`语句不需要在内存中创建临时表来完成逻辑上需要多个步骤的查询工作。

* 优化`union`查询

```sql
--高效
SELECT COL1, COL2, COL3 FROM TABLE WHERE COL1 = 10  
UNION ALL  
SELECT COL1, COL2, COL3 FROM TABLE WHERE COL3= 'TEST';
--低效
SELECT COL1, COL2, COL3 FROM TABLE WHERE COL1 = 10  
UNION  
SELECT COL1, COL2, COL3 FROM TABLE WHERE COL3= 'TEST'; 
```

如果没有 `all` 这个关键词，`MySQL` 会给临时表加上 `distinct` 选项，这会导致对整个临时表的数据做唯一性校验，这样做的消耗相当高。

* 使用 `truncate` 代替 `delete`

#### i.建表优化

* 在表中建立索引，优先考虑 `where`、`order by` 使用到的字段
* 尽量使用数字型字段
* 查询数据量大的表会造成查询缓慢

可以通过程序，分段分页进行查询，循环遍历，将结果合并处理进行展示。

* 用 `varchar`/`nvarchar` 代替 `char`/`nchar`

### 8. MySQL 基础

#### a. binlog 写入流程

<img src="http://static.gitlib.com/blog/2012/03/19/mysql1.png" alt="mysql" alt="io" style="display: block; margin: 0px auto;" />

`write`和`fsync`的写入时机，是由`sync_binlog`控制的。

* `sync_binlog=0`：每次事务提交都只 `write`，不 `fsync`；
* `sync_binlog=1`：每次事务提交都会`fsync`；
* `sync_binlog=N（N>1）`：每次提交事务都会 `write`，累计 N 个后再执行 `fsync`。

#### b. MySQL 存储引擎比较

|   存储引擎   |                            MyISAM                            |                            InnoDB                            |
| :----------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|   存储结构   | 文件名和表名相同，扩展名分别为：`.frm`（存储表定义），`.myd`（存储数据），`.myi`（存储索引） |               所有的表都保存在同一个数据文件中               |
|   事务支持   | 每次查询具有原子性,其执行数度比`InnoDB`类型更快，但是不提供事务支持。 |          提供事务支持事务，外部键等高级数据库功能。          |
|     表锁     | 只支持表级锁，用户在操作`myisam`表时，`select`，`update`，`delete`，`insert`语句都会给表自动加锁，如果加锁以后的表满足`insert`并发的情况下，可以在表的尾部插入新的数据。 | 行锁大幅度提高了多用户并发操作的新能。但是`InnoDB`的行锁，只是在`WHERE`的主键是有效的，非主键的`WHERE`都会锁全表的。 |
|   全文索引   |                支持 `FULLTEXT` 类型的全文索引                |               不支持 `FULLTEXT` 类型的全文索引               |
|    表主键    |    允许没有任何索引和主键的表存在，索引都是保存行的地址。    | 如果没有设定主键或者非空唯一索引，就会自动生成一个6字节的主键(用户不可见)，数据是主索引的一部分，附加索引保存的是主索引的值。 |
| 表的具体行数 | 保存有表的总行数，如果`select count() from table;`会直接取出出该值。 | 没有保存表的总行数，如果使用`select count() from table;`就会遍历整个表 |
|   CURD操作   |        如果执行大量的`SELECT`，`MyISAM`是更好的选择。        | 如果你的数据执行大量的`INSERT`或`UPDATE`，出于性能方面的考虑，应该使用`InnoDB`表 |
|     外键     |                            不支持                            |                             支持                             |

#### c. 主从复制

- `Slave`上面的IO进程连接上`Master`，并请求从指定日志文件的**指定位置**（或者从最开始的日志）之后的日志内容。
- `Master`接收到来自`Slave`的IO进程的请求后，负责复制的IO进程会根据请求信息读取日志指定位置之后的日志信息，返回给`Slave`的IO进程。返回信息中除了日志所包含的信息之外，还包括本次返回的信息已经到`Master`端的`bin-log`文件的名称以及`bin-log`的位置。
- `Slave`的IO进程接收到信息后，将接收到的日志内容依次添加到`Slave`端的`relay-log`文件的最末端，并将读取到的`Master`端的 `bin-log`的文件名和位置记录到`master-info`文件中，以便在下一次读取的时候能够清楚的告诉`Master`从何处开始读取日志。
- `Slave`的`Sql`进程检测到`relay-log`中新增加了内容后，会马上解析`relay-log`的内容成为在`Master`端真实执行时候的那些可执行的内容，并在自身执行。

##### 主从同步的粒度

###### [statement]()

每一条**会修改数据的sql**都会记录在`binlog`中。(一些函数无法使用`statement`进行记录，如`random()`,`sleep()`等)

###### [row]()

不记录sql语句上下文相关信息，仅保存哪条记录被修改，也就是说日志中会记录成每一行数据被修改的形式，然后在 `slave` 端再对相同的数据进行修改。

###### [mixed]()

是以上两种`level`的混合使用，一般的语句修改使用`statment`格式保存`binlog`，如一些函数，`statement`无法完成主从复制的操作，则采用`row`格式保存`binlog`。

##### 主从复制类型

###### 基于二进制日志的复制

基于`Binary log`实现的复制机制，从库需要告知主库要从哪个偏移量进行增量同步，如果指定错误会造成数据的遗漏，从而造成数据的不一致。

###### GTID 基于事务的复制

- `Master`更新数据时，会在**事务前**产生`GTID`，一同记录到`binlog`日志中；
- `Slave`在接受`Master`的`binlog`时，会校验`Master`的`GTID`是否已经执行过
- `Slave`端的I/O线程将变更的`binlog`，写入到本地的`relay log`中；
- `Sql`线程从`relay log`中获取`GTID`，然后对比`Slave`端的`binlog`是否有记录；
- 如果有记录，说明该`GTID`的事务已经执行，`slave`会忽略；
- 如果没有记录，`slave`就会从`relay log`中执行该`GTID`的事务，并记录到`binlog`；

#### d. SQL 执行原理

<img src="http://static.gitlib.com/blog/2012/06/01/sql1.png" alt="sql" alt="io" style="display: block; margin: 0px auto;" />

##### 连接器

```shell
mysql -h ip地址 -P 端口 -u 用户 -p
```

当客户端输入完了用户名和密码开始连接时，连接器会校验：

- 如果用户名或者密码不正确，客户端会收到一个`Access denied for user`的错误。
- 如果用户名和密码校验正确，连接器会检查用户所拥有的权限。之后，这个连接里的权限逻辑判断，都依赖此时读到的权限。

##### 查询缓存

`Mysql`收到一个`sql`请求之后，先检查缓存，看看之前是不是有执行过。如果执行过并缓存没有过期，结果会以`key-value`的形式存储在内存中，`key`是查询语句，`value`是查询结果。如果有缓存，直接把对应的`value`返回给客户端。

##### 分析器

词法分析代表对于你输入的多个字符加上空格组成的`sql`语句，分析器需要分析出来里面字符分别都代表什么。

语法分析将根据词法分析的结果，判断你输入的`sql`语句是否符合`MySQL`语法。

##### 优化器

优化器是在表里面有多个索引的时候，决定使用哪个索引；或者在一个语句有多表关联的时候，决定各个表的连接顺序。

##### 执行器

开始执行之前， 会先判断你对要操作的表或库有没有权限，如果没有就返回权限的错误。

如果有权限，就打开表继续执行。打开表的时候，执行器会根据表的引擎定义，去使用这个引擎提供的接口。

数据库的慢查询日志中会看到`rows_examined`的字段，表示这个语句执行过程中扫描了多少行。这个值就是在执行器每次调用引擎获取数据行的时候累加的。

#### e. InnoDB 文件存储结构

##### 表结构文件

数据库的慢查询日志中会看到`rows_examined`的字段，表示这个语句执行过程中扫描了多少行。这个值就是在执行器每次调用引擎获取数据行的时候累加的。

##### 表空间结构

<img src="http://static.gitlib.com/blog/2012/09/07/mysql10.jpg" alt="mysql" alt="io" style="display: block; margin: 0px auto;" />

###### [tablespace 表空间]()

所有数据都放在表空间中。

###### [segment 段]()

表空间由各个段组成。

###### [extent 区]()

区由连续的页组成，**在任何情况下区的大小都是1M**。InnoDB存储引擎一次从磁盘申请大概4-5个区(4-5M)。在默认情况下，页的大小为16KB，即一个区中有大概64个连续的页。

###### [page 页]()

InnoDB磁盘管理的最小单位。数据按16KB切片为Page 并编号， 编号可映射到物理文件偏移(16K * N），B+树叶子节点前后形成双向链表。

##### 缓冲池

```shell
mysql> show variables like 'innodb_buffer_pool_size';
+-------------------------+-----------+
| Variable_name           | Value     |
+-------------------------+-----------+
| innodb_buffer_pool_size | 134217728 |
+-------------------------+-----------+
1 row in set (0.00 sec)
```

缓冲池是占用最大块内存的部分，用来存放各种数据的缓存。

<img src="http://static.gitlib.com/blog/2012/09/07/mysql12.jpeg" alt="mysql" alt="io" style="display: block; margin: 0px auto;" />

##### 重做日志

日志组中的文件大小是一致的，以循环的方式运行。事务进行中时会不断的产生redo log，在事务提交时进行一次flush操作，保存到磁盘中, redo log是按照顺序写入的，磁盘的顺序读写的速度远大于随机读写。当数据库或主机失效重启时，会根据redo log进行数据的恢复，如果redo log中有事务提交，则进行事务提交修改数据。

<img src="https://s1.ax1x.com/2020/08/28/dIBxkn.png" alt="image-20200826160744839" style="display: block; margin: 0px auto;" />

写入磁盘时机：

* 主线程每秒都会将重做日志缓冲写入磁盘的重做日志文件，不论事务是否已经提交;
* 另外一个是由参数`innodb_flush_log_at_trx_commit`控制，表示在事务提交时，处理重做日志;

##### 回滚日志

`undo log`用于数据的撤回操作，它记录了修改的反向操作，比如，插入对应删除，修改对应修改为原来的数据，通过`undo log`可以实现事务回滚，并且可以根据`undo log`回溯到某个特定的版本的数据，实现`MVCC`，也即**非锁定读**。

#### f. MVCC

##### MVCC 定义

`MVCC`是通过保存数据在某个时间点的快照来实现的。不同存储引擎的`MVCC`实现是不同的，典型的有乐观并发控制和悲观并发控制。当表创建完成后，`mysql`会自动为每个表添加[数据版本号]()（最后更新数据的事务`id`）`db_trx_id` 和[删除版本号]() `db_roll_pt` （数据删除的事务id） ，事务`id`由`mysql`数据库自动生成，且递增。

##### MVCC 实现

默认的隔离级别（REPEATABLE READ）下，增删查改变成了这样：

- SELECT：读取版本号小于或等于当前事务版本号，或者删除版本为空或大于当前事务版本号的记录。这样才可以保证读取的记录是存在的。
- INSERT：将当前事务的版本号保存为行的创建版本号。
- UPDATE：新插入一行，并以当前事务的版本号作为新行的创建版本号，同时将原记录行的删除版本号设置为原来的行为版本号。
- DELETE：将当前事务的版本号保存至行的删除版本号。

#### g.索引结构

`MyISAM` & `InnoDB` 都使用B+Tree索引结构。但是底层索引存储不同，`MyISAM` 采用非聚簇索引，而`InnoDB`采用聚簇索引。

* **聚簇索引：** [索引]()和[数据文件]()为同一个文件。
* **非聚簇索引：** [索引]()和[数据文件]()分开的索引。

##### MyISAM 索引原理

采用非聚簇索引，`myi`索引文件和`myd`数据文件分离，索引文件仅保存数据记录的指针地址。叶子节点`data`域存储指向数据记录的指针地址。

<img src="http://static.gitlib.com/blog/2012/09/07/mysql15.png" alt="Mysql" style="display: block; margin: 0px auto;" />

##### InnoDB 索引原理

采用聚簇索引，`InnoDB`数据&索引文件为一个`idb`文件，表数据文件本身就是主索引，相邻的索引临近存储。

由于`InnoDB`采用聚簇索引结构存储，索`引InnoDB`的数据文件需要按照主键聚集，因此`InnoDB`要求表必须有主键。

<img src="http://static.gitlib.com/blog/2012/09/07/mysql16.png" alt="Mysql" style="display: block; margin: 0px auto;" />

##### 索引建立时机

在执行`CREATE TABLE`语句时可以创建索引，也可以单独用`CREATE INDEX`或`ALTER TABLE`来为表增加索引。

###### 1. ALTER TABLE

```sql
ALTER TABLE table_name ADD INDEX index_name (column_list)
ALTER TABLE table_name ADD UNIQUE (column_list)
ALTER TABLE table_name ADD PRIMARY KEY (column_list)
```

###### 2. CREATE INDEX

```sql
CREATE INDEX index_name ON table_name (column_list)
CREATE UNIQUE INDEX index_name ON table_name (column_list)
```

##### B+树索引

一个B+树有以下特征：

- 有n个子树的中间节点包含n个元素，每个元素不保存数据，只用来索引，所有数据都保存在叶子节点。
- 所有叶子节点包含元素的信息以及指向记录的指针，且叶子节点按关键字自小到大顺序链接。
- 所有的中间节点元素都同时存在于子节点，在子节点元素中是最大（或最小）元素。

#### h.锁

##### 加锁机制

- 乐观锁
  - 每次去取数据，都很乐观，觉得不会出现并发问题。
  - 因此，访问、处理数据每次都不上锁。
  - 但是在更新的时候，再根据版本号或时间戳判断是否有冲突，有则处理，无则提交事务。
- 悲观锁
  - 每次去取数据，很悲观，都觉得会被别人修改，会有并发问题。
  - 因此，访问、处理数据前就加**排他锁**。
  - 在整个数据处理过程中锁定数据，事务提交或回滚后才释放锁。

##### 锁粒度

- 表锁： 开销小，加锁快；锁定力度大，发生锁冲突概率高，并发度最低;不会出现死锁。
- 行锁： 开销大，加锁慢；会出现死锁；锁定粒度小，发生锁冲突的概率低，并发度高。
- 页锁： 开销和加锁速度介于表锁和行锁之间；会出现死锁；锁定粒度介于表锁和行锁之间，并发度一般。

##### 兼容性

- 共享锁
  - 又称读锁（S锁）。
  - 一个事务获取了共享锁，其他事务可以获取共享锁，不能获取排他锁，其他事务可以进行读操作，不能进行写操作。
  - SELECT … LOCK IN SHARE MODE 显示加共享锁。
- 排他锁
  - 又称写锁（X锁）。
  - 如果事务T对数据A加上排他锁后，则其他事务不能再对A加任任何类型的封锁。获准排他锁的事务既能读数据，又能修改数据。
  - SELECT … FOR UPDATE 显示添加排他锁。

##### 锁模式

- 记录锁：在行相应的索引记录上的锁，锁定一个行记录
- gap锁：是在索引记录间歇上的锁,锁定一个区间
- next-key锁：是记录锁和在此索引记录之前的gap上的锁的结合，锁定行记录+区间。
- 意向锁：是为了支持多种粒度锁同时存在；

#### i.复制机制

##### 异步复制

MySQL默认的复制即是异步的，主库在执行完客户端提交的事务后会立即将结果返给给客户端，并不关心从库是否已经接收并处理，这样就会有一个问题，主如果crash掉了，此时主上已经提交的事务可能并没有传到从上，如果此时，强行将从提升为主，可能导致新主上的数据不完整。

##### 同步复制

同步复制指当主库执行完一个事务，所有的从库都执行了该事务才返回给客户端。因为需要等待所有从库执行完该事务才能返回，所以全同步复制的性能必然会收到严重的影响。

##### 半同步复制

MySQL从5.5开始推出了半同步复制。相比异步复制，半同步复制提高了数据完整性，因为很明确知道，在一个事务提交成功之后，这个事务就至少会存在于两个地方。

###### [半同步复制原理]()

- 从库会在连接到主库时告诉主库，它是不是配置了半同步。
- 如果半同步复制在主库端是开启了的，并且至少有一个半同步复制的从库节点，那么此时主库的事务线程在提交时会被阻塞并等待，结果有两种可能，要么至少一个从库节点通知它已经收到了所有这个事务的Binlog事件，要么一直等待直到超过配置的某一个时间点为止，而此时，半同步复制将自动关闭，转换为异步复制。
- 从库节点只有在接收到某一个事务的所有Binlog，将其写入并Flush到Relay Log文件之后，才会通知对应主库上面的等待线程。
- 如果在等待过程中，等待时间已经超过了配置的超时时间，没有任何一个从节点通知当前事务，那么此时主库会自动转换为异步复制，当至少一个半同步从节点赶上来时，主库便会自动转换为半同步方式的复制。
- 半同步复制必须是在主库和从库两端都开启时才行，如果在主库上没打开，或者在主库上开启了而在从库上没有开启，主库都会使用异步方式复制。

###### [AFTER_COMMIT模式]()

![Mysql](http://static.gitlib.com/blog/2017/01/23/mysql2.jpeg)

###### [AFTER_SYNC模式]()

![Mysql](http://static.gitlib.com/blog/2017/01/23/mysql3.jpeg)

#### j. ACID特性

* 事务的原子性是通过 undo log 来实现的
* 事务的持久性性是通过 redo log 来实现的
* 事务的隔离性是通过（读写锁+MVCC）来实现的
* 事务的一致性是通过原子性，持久性，隔离性来实现的

#### k. 分库分表

##### 分表

###### 垂直分表

垂直分表适合表中存在不常用并且占用了大量空间的表拆分出去。垂直分表影响就是之前只要一个查询的，现在需要两次查询才能拿到分表之前的完整用户信息。

###### 水平分表

水平分表就是将同一张表中的数据，才拆分到同样表结构。

1. 按范围分表
2. 按Hash值取模分表
3. 新增路由关系表

##### 分库

分表完成后可以解决单表的压力，但数据库本身的压力却没有下降，数据库里“其他表”的写入依然导致整个数据库 IO 增加，可以尝试把“业务关联度”不是很高的表单独移到一个新的数据库中，完全和现有的业务隔离开来。

### 9. TCP/IP 四层结构，每层有哪些协议

![image-20200724095019093](https://s1.ax1x.com/2020/08/21/dYaucT.png)

<img src="https://img-blog.csdn.net/20180930155137505?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NzZG5fa291/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70" alt="TCP/IP四层协议" style="zoom:150%;" />

#### a. 数据链路层

* `ARP`（Address Resolve Protocol，地址解析协议）

  网络层必须先将目标机器的IP地址转化成其物理地址，才能使用数据链路层提供的服务。

* `RARP`（Reverse Address Resolve Protocol，逆地址解析协议）

  运行`RARP`服务的网络管理者通常存有该网络上所有机器的物理地址到IP地址的映射。

#### b. 网络层

* `IP`（Internet Protocol，因特网协议）

  IP协议使用逐跳（hop by hop）的方式确定通信路径。

* `ICMP`（Internet Control Message Protocol，因特网控制报文协议）

  主要用于检测网络连接。

  * 差错报文：用来回应网络错误，比如目标不可达（3）和重定向（5）；
  * 查询报文：查询网络信息，比如ping程序就是使用ICMP报文查看目标是否可到达（8）的。

  ICMP报文使用16位校验和字段对整个报文（包括头部和内容部分）进行循环冗余校验（Cyclic Redundancy Check，CRC），以检验报文在传输过程中是否损坏。不同的ICMP报文类型具有不同的正文内容。

#### c. 传输层

* TCP（Transmission Control Protocol，传输控制协议）

  为应用层提供可靠的、面向连接的和基于流（stream）的服务。TCP协议使用超时重传、数据确认等方式来确保数据包被正确地发送至目的端。TCP服务是基于流的。基于流的数据没有边界（长度）限制，它源源不断地从通信的一端流入另一端。发送端可以逐个字节地向数据流中写入数据，接收端也可以逐个字节地将它们读出。

* UDP（User Datagram Protocol，用户数据报协议）

  为应用层提供不可靠、无连接和基于数据报的服务。基于数据报的服务，是相对基于流的服务而言的。每个UDP数据报都有一个长度，接收端必须以该长度为最小单位将其所有内容一次性读出，否则数据将被截断。

### 10. DNS服务器原理

#### a. DNS解析过程

![img](https://pic1.zhimg.com/80/v2-30bdd430e5066476a87f977009f990cf_720w.jpg)

#### b. DNS什么时候使用TCP，什么时候使用UDP

DNS在域名解析过程中使用UDP，在主DNS服务器与辅助DNS服务器之间通信，并加载数据信息（又称为区域复制）过程中使用TCP。

* 域名解析使用UDP

  因为UDP块

* 区域传送使用TCP

  因为TCP可靠

#### c. DNS原理

```shell
hdc@hdc:~$ dig www.tmall.com

; <<>> DiG 9.11.3-1ubuntu1.12-Ubuntu <<>> www.tmall.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 39935
;; flags: qr rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;www.tmall.com.                 IN      A

;; ANSWER SECTION:
www.tmall.com.          201     IN      CNAME   www.tmall.com.danuoyi.tbcache.com.
www.tmall.com.danuoyi.tbcache.com. 30 IN A      123.235.33.104
www.tmall.com.danuoyi.tbcache.com. 30 IN A      123.235.33.105
www.tmall.com.danuoyi.tbcache.com. 30 IN A      111.164.16.233

;; Query time: 13 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
;; WHEN: Thu Aug 20 16:40:16 CST 2020
;; MSG SIZE  rcvd: 134
```

`www.tmall.com`对应的真正的域名为`www.tmall.com.`。末尾的`.`称为根域名，因为每个域名都有根域名，因此我们通常省略。

分级查询流程：

1. 先在本机的DNS里头查，如果有就直接返回了。
2. 本机DNS里头发现没有，就去根服务器里查。
3. 本机的DNS接到又会向`com`域的DNS服务器发送查询消息。
4. 重复前面的步骤，就可以找到目标DNS服务器。

### 11. 线程的五种状态及转换

#### a. 线程5种状态（JAVA）

* 新建状态
* 就绪状态
* 运行状态
* 阻塞状态
  * 等待阻塞：`wait()`
  * 同步阻塞：请求获取同步锁时，该同步锁被别的线程占用
  * 其他阻塞：`sleep()`或`join()`
* 死亡状态

<img src="http://img.blog.csdn.net/20140828202610671" alt="img" style="zoom:150%;" />

#### b. 协程6种状态（Go）

![img](https://static.studygolang.com/171207/80c5272a28b7b5ed5a1bf3f24157dd27.jpg)

* 初始化状态（只是分配的地址，但是并未
* 就绪状态
* 运行状态
* 系统调用状态（保证高的并发性能）
* 阻塞状态
* 死亡状态

#### c. 线程与进程之间的资源的关系

一个进程由一到多个线程组成，各线程共享进程的内存空间（代码，数据，堆）和一些进程级的资源（打开的文件和信号）。进程有自己独立的寄存器和栈。

线程私有的是：局部变量，函数的参数，TLS（Thread Local Storage，线程局部存储）数据。

线程之间共享（进程所有）：全局变量，堆，函数里的静态变量，程序代码，打开的文件。

### 12. 动态链接库与静态链接库的区别

都属于共享代码的方式。

* 静态链接

动态链接的基本思想是把程序按照模块拆分成各个相对独立部分，在程序运行时才将它们链接在一起形成一个完整的程序，而不是像静态链接一样把所有程序模块都链接成一个单独的可执行文件。
共享库：就是即使需要每个程序都依赖同一个库，但是该库不会像静态链接那样在内存中存在多分，副本，而是这多个程序在执行时共享同一份副本；
更新方便：更新时只需要替换原来的目标文件，而无需将所有的程序再重新链接一遍。当程序下一次运行时，新版本的目标文件会被自动加载到内存并且链接起来，程序就完成了升级的目标。
性能损耗：因为把链接推迟到了程序运行时，所以每次执行程序都需要进行链接，所以性能会有一定损失。

* 静态链接

函数和数据被编译进一个二进制文件。在使用静态库的情况下，在编译链接可执行文件时，链接器从库中复制这些函数和数据并把它们和应用程序的其它模块组合起来创建最终的可执行文件。
空间浪费：因为每个可执行程序中对所有需要的目标文件都要有一份副本，所以如果多个程序对同一个目标文件都有依赖，会出现同一个目标文件都在内存存在多个副本；
更新困难：每当库函数的代码修改了，这个时候就需要重新进行编译链接形成可执行程序。
运行速度快：但是静态链接的优点就是，在可执行程序中已经具备了所有执行程序所需要的任何东西，在执行的时候运行速度快。

### 12. CPU多级缓存

<img src="https://img2018.cnblogs.com/blog/292888/201904/292888-20190405102602605-80698397.png" alt="img" style="zoom:150%;" />

级别越小的缓存，越接近CPU， 意味着速度越快且容量越少。

`L1`是最接近`CPU`的，它容量最小，速度最快，每个核上都有一个`L1 Cache`(准确地说每个核上有两个`L1 Cache`， 一个存数据 `L1d Cache`， 一个存指令 `L1i Cache`)。

`L2 Cache` 更大一些，例如`256K`，速度要慢一些，一般情况下每个核上都有一个独立的`L2 Cache`；二级缓存就是一级缓存的缓冲器：一级缓存制造成本很高因此它的容量有限，二级缓存的作用就是存储那些`CPU`处理时需要用到、一级缓存又无法存储的数据。

`L3 Cache`是三级缓存中最大的一级，例如`12MB`，同时也是最慢的一级，在同一个`CPU`插槽之间的核共享一个`L3 Cache`。三级缓存和内存可以看作是二级缓存的缓冲器，它们的容量递增，但单位制造成本却递减。

| 从CPU到  | 大约需要的CPU周期 | 大约需要的时间（单位ns） |
| :------: | :---------------: | :----------------------: |
|  寄存器  |      1cycle       |                          |
| L1 Cache |    ~3-4 cycles    |        ~0.5-1 ns         |
| L2 Cache |   ~10-20 cycles   |         ~3-7 ns          |
| L3 Cache |   ~40-45 cycles   |          ~15 ns          |
| 跨槽传输 |                   |          ~20 ns          |
|   内存   |  ~120-240 cycles  |        ~60-120ns         |

#### a. cache 缓存带来的问题

导致数据不一致的问题，对于多核系统来说。

#### b. 缓存一致性

在MESI协议中，每个cache line有4个状态，可用2个bit表示，它们分别是：

| 状态           | 描述                                                         |
| -------------- | ------------------------------------------------------------ |
| M（Modified）  | 这行数据有效，数据被修改了，和内存中的数据不一致，数据只存在于本Cache中。 |
| E（Exclusive） | 这行数据有效，数据和内存中的数据一致，数据只存在于本Cache中。 |
| S（Shared）    | 这行数据有效，数据和内存中的数据一致，数据存在于很多Cache中。 |
| I（Invalid）   | 这行数据无效                                                 |

* E状态

![E状态](https://s1.ax1x.com/2020/08/21/dYak7j.png)

* S状态

![S状态](https://s1.ax1x.com/2020/08/21/dYaFBQ.png)

* M与I状态

![M状态与I状态](https://s1.ax1x.com/2020/08/21/dYaEAs.png)

* I状态的迁移过程

| 事件         | 行为                                                         | 下一个状态 |
| ------------ | ------------------------------------------------------------ | ---------- |
| Local Read   | 如果其他Cache中没有这份数据，本Cache从内存中取数据，Cache line状态变成E；如果其他Cache有数据，且状态为M，则先要更新到内存，再从内存中取，两个Cache的Cache line 同时变成S；如果其它Cache有这份数据，状态为S或者E，本Cache从内存中取数据，这些Cache的Cache line状态都变成S | E/S        |
| Local Write  | 从内存中取数据，在Cache中修改，状态变成M；如果其他Cache有这份数据，且状态为M，则要先将数据更新到内存；如果其他Cache有这份数据，则其它Cache的Cache line 状态变成I | M          |
| Remote Read  | 别的核的操作与它无关                                         | I          |
| Remote Write | 别的核的操作与它无关                                         | I          |

* E状态的迁移过程

| 事件         | 行为                                           | 下一个状态 |
| ------------ | ---------------------------------------------- | ---------- |
| Local Read   | 从Cache中取数据，状态不变                      | E          |
| Local Write  | 修改Cache中的数据，状态变成M                   | M          |
| Remote Read  | 数据和其他内核共用，状态变成了S                | S          |
| Remote Write | 数据被修改，本Cache line 不能再使用，状态变成I | I          |

* S状态的迁移过程

| 事件         | 行为                                                         | 下一个状态 |
| ------------ | ------------------------------------------------------------ | ---------- |
| Local Read   | 从Cache中取数据，状态不变                                    | S          |
| Local Write  | 修改Cache中的数据，状态变成M，其他内核共享的Cache line 状态变成 I | M          |
| Remote Read  | 状态不变                                                     | S          |
| Remote Write | 数据被修改，本Cache line 不能再使用，状态变成I               | I          |

* M状态的迁移过程

| 事件         | 行为                                                         | 下一个状态 |
| ------------ | ------------------------------------------------------------ | ---------- |
| Local Read   | 从Cache中取数据，状态不变                                    | M          |
| Local Write  | 修改Cache中的数据，状态不变                                  | M          |
| Remote Read  | 这行数据被写到内存中，其他内核能使用到最新的数据，状态变成S  | S          |
| Remote Write | 这行数据被写到内存中，其他内核能使用到最新的数据，由于其他内核会修改这行数据，状态变成I | I          |

![image-20200820233558286](https://s1.ax1x.com/2020/08/21/dYam90.png)

### 14. 三次握手与四次挥手

#### a. TCP

- TCP 提供一种**面向连接的、可靠的**字节流服务
- 在一个 TCP 连接中，仅有两方进行彼此通信。广播和多播不能用于 TCP
- TCP 使用校验和，确认和重传机制来保证可靠传输
- TCP 给数据分节进行排序，并使用累积确认保证数据的顺序不变和非重复
- TCP 使用滑动窗口机制来实现流量控制，通过动态改变窗口的大小进行拥塞控制

#### b. 三次握手

客户端执行`connect()`，触发三次握手。

* 第一次握手（SYN=1，seq=x）

  客户端发送，并进入`SYN_SEND`状态。

> x即ISN（Initial Sequence Number），随时间而变化，是一个32比特的计数器，每4ms加1。

* 第二次握手（SYN=1，ACK=1，seq=y，ACKnum=x+1）

  服务端发送，并进入`SYN_RCVD`状态。

> 此时双方还没有完全建立连接，服务器会把此种状态下请求连接放在一个队列里，将该队列称之为半连接队列。

* 第三次握手（ACK=1，ACKnum=y+1）

  客户端发送，并进入`ESTABLISHED`状态。

> 在第三次握手时是可以携带数据的。

![three-way-handshake](https://raw.githubusercontent.com/HIT-Alibaba/interview/master/img/tcp-connection-made-three-way-handshake.png)

> SYN攻击是Client在短时间内伪造大量不存在的IP地址，并向Server不断地发送SYN包，Server则回复确认包，并等待Client确认，由于源地址不存在，因此Server需要不断重发直至超时，这些伪造的SYN包将长时间占用未连接队列，导致正常的SYN请求因为队列满而被丢弃，从而引起网络拥塞甚至系统瘫痪。
>
> 检测方法：当服务器上看到大量的半连接状态时，特别是源IP地址是随机的，基本上可以断定这是一次SYN攻击。
>
> ```shell
> netstat -n -p TCP | grep SYN_RECV
> ```

#### c. 四次挥手

任何一方执行`close()`操作即可产生挥手操作。

* 第一次挥手（FIN=1，seq=x）

  发送完毕后，进入`FIN_WAIT_1`状态。

* 第二次挥手（ACK=1，ACKnum=x+1）

  发送完毕后，发送方进入`CLOSE_WAIT`状态，接收方进入`FIN_WAIT_2`状态。

* 第三次挥手（FIN=1，seq=y）

  发送完毕后，发送方进入`LAST_ACK`状态。

* 第四次挥手（ACK=1，ACKnum=y+1）

  发送完毕后，进入`TIME_WAIT`状态，等待可能出现的要求重传的`ACK`包。

> 客户端等待了某个固定时间（两个最大段生命周期，2MSL，2 Maximum Segment Lifetime）之后，没有收到服务器端的 ACK，认为服务器端已经正常关闭连接，于是自己也关闭连接，进入 `CLOSED` 状态。

![four-way-handshake](https://raw.githubusercontent.com/HIT-Alibaba/interview/master/img/tcp-connection-closed-four-way-handshake.png)

> 出现大量 `TIME_WAIT`的原因？
>
> 在**高并发短连接**的TCP服务器上，当服务器处理完请求后立刻主动正常关闭连接。这个场景下会出现大量socket处于TIME_WAIT状态。如果客户端的并发量持续很高，此时部分客户端就会显示连接不上。
>
> 如何避免出现大量`TIME_WAIT？
>
> 1. 服务器可以设置`SO_REUSEADDR`套接字选项来通知内核，如果端口忙，但TCP连接位于TIME_WAIT状态时可以重用端口。
> 2. 在设计协议逻辑的时候，尽量由客户端主动关闭，避免服务端出现`TIME_WAIT`。
> 3. 将频繁创建的短链接改成长连接。

> TCP中的Keep-alive保活机制的基本原理是，隔一段时间给连接对端发送一个探测包，如果收到对方回应的 ACK，则认为连接还是存活的，在超过一定重试次数之后还是没有收到对方的回应，则丢弃该 TCP 连接。
>
> 但次方法并不能保证服务的可用，还需要在应用层添加自己的心跳功能。

### 15. 浏览器URL的请求过程

* DNS解析
* TCP连接
* 发送HTTP请求
* 服务器处理请求并返回HTTP报文
* 浏览器解析并渲染页面
* 连接结束

![image-20200821101209174](https://s1.ax1x.com/2020/08/21/dYaZhq.png)

#### a. GET与POST的区别

* GET请求在URL中传送的参数是有长度限制的，而POST没有。
* GET比POST更不安全，因为参数直接暴露在URL上，所以不能用来传递敏感信息。
* GET参数通过URL传递，POST放在Request body中。
* GET请求参数会被完整保留在浏览器历史记录里，而POST中的参数不会被保留。
* GET请求只能进行url编码，而POST支持多种编码方式。
* GET请求会被浏览器主动cache，而POST不会，除非手动设置。
* GET产生的URL地址可以被Bookmark，而POST不可以。
* GET在浏览器回退时是无害的，而POST会再次提交请求。

> GET和POST还有一个重大区别，简单的说：
>
> GET产生一个TCP数据包；POST产生两个TCP数据包。
>
> 长的说：
>
> 对于GET方式的请求，浏览器会把http header和data一并发送出去，服务器响应200（返回数据）；
>
> 而对于POST，浏览器先发送header，服务器响应100 continue，浏览器再发送data，服务器响应200 ok（返回数据）。

#### b. HTTP状态码

* 临时响应

  表示临时响应并需要请求者继续执行操作的状态码。

  * `100(继续)`请求者应当继续提出请求。服务器返回此代码表示已收到请求的第一部分，正在等待其余部分。

  * `101(切换协议)`请求者已要求服务器切换协议，服务器已确认并准备切换。

* 成功

  表示成功处理了请求的状态码。

  * `200(成功)`服务器已成功处理了请求。
  * `201(已创建)`请求成功并且服务器创建了新的资源。
  * `202(已接受)`服务器已接受请求，但尚未处理。
  * `203(非授权信息)`服务器已成功处理了请求，但返回的信息可能来自另一来源。
  * `204(无内容)`服务器成功处理了请求，但没有返回任何内容。
  * `205(重置内容)`服务器成功处理了请求，但没有返回任何内容。
  * `206(部分内容)`服务器成功处理了部分 GET 请求。

* 重定向

  * `300(多种选择)`针对请求，服务器可执行多种操作。
  * `301(永久移动)`请求的网页已永久移动到新位置。
  * `302(临时移动)`服务器目前从不同位置的网页响应请求，但请求者应继续使用原有位置来响应以后的请求。
  * `303(查看其他位置)`请求者应当对不同的位置使用单独的 GET 请求来检索响应时，服务器返回此代码。
  * `304(未修改)`自从上次请求后，请求的网页未修改过。
  * `305(使用代理)`请求者只能使用代理访问请求的网页。

* 请求错误

  这些状态码表示请求可能出错，妨碍了服务器的处理。

  * `400(错误请求)`服务器不理解请求的语法。
  * `401(未授权)`请求要求身份验证。
  * `403(禁止)`服务器拒绝请求。
  * `404(未找到)`服务器找不到请求的网页。
  * `405(方法禁用)`禁用请求中指定的方法。
  * `406(不接受)`无法使用请求的内容特性响应请求的网页。
  * `408(请求超时)`服务器等候请求时发生超时。
  * `411(需要有效长度)`服务器不接受不含有效内容长度标头字段的请求。
  * `412(未满足前提条件)`服务器未满足请求者在请求中设置的其中一个前提条件。
  * `413(请求实体过大)`服务器无法处理请求，因为请求实体过大，超出服务器的处理能力。
  * `414(请求的 URI 过长)`请求的 URI(通常为网址)过长，服务器无法处理。
  * `415(不支持的媒体类型)`请求的格式不受请求页面的支持。

* 服务器错误

  这些状态码表示服务器在处理请求时发生内部错误。

  * `500(服务器内部错误)`服务器遇到错误，无法完成请求。
  * `501(尚未实施)`服务器不具备完成请求的功能。
  * `502(错误网关)`服务器作为网关或代理，从上游服务器收到无效响应。
  * `503(服务不可用)`服务器目前无法使用(由于超载或停机维护)。
  * `504(网关超时)`服务器作为网关或代理，但是没有及时从上游服务器收到请求。
  * `505(HTTP 版本不受支持)`服务器不支持请求中所用的 HTTP 协议版本。

#### c. HTTP报文

一个HTTP请求报文由请求行（request line）、请求头部（header）、空行和请求数据4个部分组成。

![img](https://pic002.cnblogs.com/images/2012/426620/2012072810301161.png)

### 16. Cookies，Sessions，JWT的区别

#### a. Cookies

**HTTP 是无状态的协议（对于事务处理没有记忆能力，每次客户端和服务端会话完成时，服务端不会保存任何会话信息）。**

**cookie 存储在客户端**： cookie 是服务器发送到用户浏览器并保存在本地的一小块数据，它会在浏览器下次向同一服务器再发起请求时被携带并发送到服务器上。

#### b. Session

- **session 是另一种记录服务器和客户端会话状态的机制**
- **session 是基于 cookie 实现的，session 存储在服务器端，sessionId 会被存储到客户端的cookie 中**

![Session认证流程](https://img-blog.csdnimg.cn/20200813163907830.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0h1bnRlcm1hbndw,size_16,color_FFFFFF,t_70#pic_center)

> Cookie与Session的区别
>
> * 安全性：Session 比 Cookie 安全，Session 是存储在服务器端的，Cookie 是存储在客户端的。
> * 存储值的类型不同：Cookie 只支持存字符串数据，想要设置其他类型的数据，需要将其转换成字符串，Session 可以存任意数据类型。
> * 有效期不同： Cookie 可设置为长时间保持，比如我们经常使用的默认登录功能，Session 一般失效时间较短，客户端关闭（默认情况下）或者 Session 超时都会失效。
> * 存储大小不同：单个 Cookie 保存的数据不能超过 4K，Session 可存储数据远高于 Cookie，但是当访问量过多，会占用过多的服务器资源。

#### c. JWT

是一种**认证授权机制**。JWT 的声明一般被用来在身份提供者和服务提供者间传递被认证的用户身份信息，以便于从资源服务器获取资源。比如用在用户登录上。可以使用 HMAC 算法或者是 RSA 的公/私秘钥对 JWT 进行签名。

![JWT原理](https://img-blog.csdnimg.cn/2020081409080860.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0h1bnRlcm1hbndw,size_16,color_FFFFFF,t_70#pic_center)

![img](https://www.wangbase.com/blogimg/asset/201807/bg2018072303.jpg)

* JWT 默认是不加密，但也是可以加密的。生成原始 Token 以后，可以用密钥再加密一次。
* JWT 不仅可以用于认证，也可以用于交换信息。有效使用 JWT，可以降低服务器查询数据库的次数。
* JWT 本身包含了认证信息，一旦泄露，任何人都可以获得该令牌的所有权限。为了减少盗用，JWT的有效期应该设置得比较短。对于一些比较重要的权限，使用时应该再次对用户进行认证。
* JWT 适合一次性的命令认证，颁发一个有效期极短的 JWT，即使暴露了危险也很小，由于每次操作都会生成新的 JWT，因此也没必要保存 JWT，真正实现无状态。
  为了减少盗用，JWT 不应该使用 HTTP 协议明码传输，要使用 HTTPS 协议传输。

### 17. Token Buckets

<img src="https://s1.ax1x.com/2020/08/28/doP5E6.png" alt="image-20200825090540242" style="zoom:67%;" />

#### a.Token Buckets 原理

1. 首先设有一个令牌桶，桶内存放令牌，一开始令牌桶内的令牌是满的（桶内令牌的数量可根据服务器情况设定）;
2. 每次访问从桶内取走一个令牌，当桶内令牌为0，则不允许再访问;
3. 每隔一段时间，再放入令牌，最多使桶内令牌满额。（可以根据实际情况，每隔一段时间放入若干个令牌，或直接补满令牌桶）;

#### b.使用Redis的队列实现令牌桶（Redis的List实现）

```php
<?php

class TokenBucket {
    private $_config;
    private $_redis;
    private $_queue;
    private $_max;
    
    /**
     * 初始化
     * @param Array $config redis连接设定
     */
    public function __construct($config, $queue, $max){
        $this->_config = $config;
        $this->_queue = $queue;
        $this->_max = $max;
        $this->_redis = $this->connect();
    }
    /**
     * 加入令牌
     * @param  Int $num 加入的令牌数量
     * @return Int 加入的数量
     */
    public function add($num=0) {
        // 当前剩余令牌数
        $curnum = intval($this->_redis->lSize($this->_queue));
        // 最大令牌数
        $maxnum = intval($this->_max);
        // 计算最大可加入的令牌数量，不能超过最大令牌数
        $num = $maxnum >= $curnum + $num? $num : $maxnum - $curnum;
        // 加入令牌
        if($num <0) {
            $token = array_fill(0, $num, 1);
            $this->_redis->lPush($this->_queue, ...$token);
            return $num;
        }
        return 0;
    }
    
    /**
     * 获取令牌
     * @return Boolean
     */
    public function get(){
        return $this->_redis->rPop($this->_queue)? true : false;
    }
	
    /**
     * 重设令牌桶，填满令牌
     */
    public function reset(){
        $this->_redis->delete($this->_queue);
        $this->add($this->_max);
    }
    
    /**
     * 创建redis连接
     * @return Link
     */
    private function connect(){
        try{
            $redis = new Redis();
            $redis->connect($this->_config['host'],$this->_config['port']);
        }catch(RedisException $e){
            throw new Exception($e->getMessage());
            return false;
        }
        return $redis;
    }
}
?>
```

主要使用的是`Redis`中的`List`实现，使用的命令有`lPush`与`rPop`。

#### c. Redis 中 List 基本实现原理

**ziplist**

`ziplist`使用局限性：字段、值比较小，才会用`ziplist`。

完整的`zlentry`（`ziplist`内部组成）由以下3各部分组成：

- `prevrawlen`：记录前一个节点所占有的内存字节数，通过该值，我们可以从当前节点计算前一个节点的地址，可以用来实现表尾向表头节点遍历；

- `len/encoding`：记录了当前节点content占有的内存字节数及其存储类型，用来解析content用；

- `content`：保存了当前节点的值。

最关键的是`prevrawlen`和`len/encoding`，`content`只是实际存储数值的比特位。

**`prevrawlen`是变长编码，有两种表示方法：**

- 如果前一节点的长度小于 254 字节，则使用1字节(`uint8_t`)来存储`prevrawlen`；
- 如果前一节点的长度大于等于 254 字节，那么将第 1 个字节的值设为 254 ，然后用接下来的 4 个字节保存实际长度。

`ziplist`的主要优点是节省内存，但它上面的查找操作只能按顺序查找。

**quickList**

![img](https://pic2.zhimg.com/80/v2-800dbf77bba29897de1ad769d0149f8f_720w.jpg)

`quickList`就是一个标准的双向链表的配置，有`head` 有`tail`;
每一个节点是一个`quicklistNode`，包含`prev`和`next`指针。
每一个`quicklistNode` 包含 一个`ziplist`，`*zp` 压缩链表里存储键值。
所以`quicklist`是对`ziplist`进行一次封装，使用小块的`ziplist`来既保证了少使用内存，也保证了性能。

#### d. Redis 哨兵机制

##### 1. 故障转移

1. 每个Sentinel以每秒钟一次的频率向它所知的Master，Slave以及其他 Sentinel 实例发送一个 PING 命令。
2. 如果一个实例（instance）距离最后一次有效回复 PING 命令的时间超过 down-after-milliseconds 选项所指定的值， 则这个实例会被当前 Sentinel 标记为主观下线。
3. 如果一个Master被标记为主观下线，则正在监视这个Master的所有 Sentinel 要以每秒一次的频率确认Master的确进入了主观下线状态。
4. 当有足够数量的 Sentinel（大于等于配置文件指定的值）在指定的时间范围内确认Master的确进入了主观下线状态， 则Master会被标记为客观下线 。
5. 当Master被 Sentinel 标记为客观下线时，Sentinel 向下线的 Master 的所有 Slave 发送 INFO 命令的频率会从 10 秒一次改为每秒一次 （在一般情况下， 每个 Sentinel 会以每 10 秒一次的频率向它已知的所有Master，Slave发送 INFO 命令 ）。
6. 若没有足够数量的 Sentinel 同意 Master 已经下线， Master 的客观下线状态就会变成主观下线。若 Master 重新向 Sentinel 的 PING 命令返回有效回复， Master 的主观下线状态就会被移除。
7. `sentinel`节点会与其他`sentinel`节点进行“沟通”，投票选举一个sentinel节点进行故障处理，在从节点中选取一个主节点，其他从节点挂载到新的主节点上自动复制新主节点的数据。

##### 2. Master选举的标准

**（1）跟master断开连接的时长。**
如果一个slave跟master断开连接已经超过了down-after-milliseconds的10倍，外加master宕机的时长，那么slave就被认为不适合选举为master.

```shell
( down-after-milliseconds * 10) + milliseconds_since_master_is_in_SDOWN_state
```

**（2）slave优先级。**
按照slave优先级进行排序，slave priority越低，优先级就越高

**（3）复制offset。**
如果slave priority相同，那么看replica offset，哪个slave复制了越多的数据，offset越靠后，优先级就越高

**（4）run id**
如果上面两个条件都相同，那么选择一个run id比较小的那个slave

##### 3. AOF 与 RDB

`AOF`，记录每次写请求的命令，以追加的方式在文件尾部追加，效率比较高。
对于操作系统来说，不是每次写都直接写到磁盘，操作系统自己会有一层cache，`redis`写磁盘的数据会先缓存在`os cache`里，`redis`每隔1秒调用一次操作系统的`fsync`操作，强制将`os cache`中的数据刷入`AOF`文件中。

当`redis`重启的时候，就把`AOF`中记录的命令重新执行一遍就可以了，但是如果文件很大的话，执行会耗费较多的时间，对于数据恢复来说耗时会多一点。

`RDB`，是快照文件，每隔一定时间将`redis`内存中的数据生成一份完整的`RDB`快照文件，当`redis`重启的时候直接加载数据即可，同样的数据比`AOF`恢复的要快。

> 至于，`RDB`和`AOF`到底该如何选择，我觉得两种都选择：
>
> 1. 不要仅仅使用`RDB`，因为那样会导致你丢失很多数据。
>
> 2. 也不要仅仅使用AOF，因为那样有两个问题，
>
>    第一，你通过AOF做冷备，没有RDB做冷备，来的恢复速度更快;
>
>    第二，RDB每次简单粗暴生成数据快照，更加健壮，可以避免AOF这种复杂的备份和恢复机制的bug。
>
> 3. 综合使用AOF和RDB两种持久化机制，用AOF来保证数据不丢失，作为数据恢复的第一选择; 用RDB来做不同程度的冷备，在AOF文件都丢失或损坏不可用的时候，还可以使用RDB来进行快速的数据恢复，作为数据恢复的最后一道防线。

### 19. Redis

#### a.异步消息队列

<img src="https://s1.ax1x.com/2020/08/28/doiC8g.png" alt="image-20200826165624738" style="zoom:80%;" />

使用`rpush/lpush`操作入队列，使用`rpop/lpop`操作出队列。

为了避免直接睡眠导致的消息延迟增大问题，阻塞读在队列没有数据的时候，会立即进入休眠状态，一旦数据到来，则立刻醒过来。消息的延迟几乎为零。用`blpop/brpop` 替代前面的 `lpop/rpop`，就完美解决了上面的问题。

```shell
127.0.0.1:6379> rpush notify-queue apple banana pear
(integer) 3
127.0.0.1:6379> llen notify-queue
(integer) 3
127.0.0.1:6379> lpop notify-queue
"apple"
127.0.0.1:6379> blpop notify-queue 3
1) "notify-queue"
2) "banana"
127.0.0.1:6379> blpop notify-queue 3
1) "notify-queue"
2) "pear"
127.0.0.1:6379> blpop notify-queue 3
(nil)
(3.09s)
127.0.0.1:6379>
```

#### b.延时队列

延时队列可以通过 Redis 的 zset(有序列表) 来实现。我们将消息序列化成一个字符串作为 zset 的value，这个消息的到期处理时间作为score，然后用多个线程轮询 zset 获取到期的任务进行处理，多个线程是为了保障可用性，万一挂了一个线程还有其它线程可以继续处理。因为有多个线程，所以需要考虑并发争抢任务，确保任务不能被多次执行。 

```go
import "github.com/garyburd/redigo/redis"
// 添加一个元素
func AddZset(data, zsetName string, score int64) (err error) {
    con := pool.Get()
    defer cn.Close()
    _, err = con.Do("zadd", zsetname, score, data)
    if err != nil {
        return
    }
    return 
}
// 按照score值从小到大遍历zset，提供start和end两个下标参数
func RangeZset(start, end int, zsetName string) (data []string, err error) {
    con := pool.Get()
    defer con.Close()
    data, err = redis.Strings(con.Do("zrange", zsetName, start, end, "withscores"))
    return
}
// 它会返回zset中介于min和max之间的所有元素
func RangeZsetByScore(start, end int64, zsetName string) (data []string, err error) {
	con := pool.Get()
	defer con.Close()
	data, err = redis.Strings(con.Do("zrangebyscore", zsetName, start, end, "withscores"))
	return
}
// 需要将其删除
func RemZset(zsetName string, keys []string) (err error) {
	if len(keys) == 0 {
		return
	}
	con := pool.Get()
	defer con.Close()
	_, err = con.Do("zrem", redis.Args{}.Add(zsetName).AddFlat(keys)...)
	return
}
```

#### c.发布/订阅

消息发布者，**无需独占链接**，你可以在publish消息的同时，使用同一个redis-client链接进行其他操作。

消息订阅者，**需要独占链接**，即进行subscribe期间，redis-client无法穿插其他操作，此时client以阻塞的方式等待“publish端”的消息；因此这里subscribe端需要使用单独的链接，甚至需要在额外的线程中使用。

TCP默认连接时间固定，如果在这时间内sub端没有接收到pub端消息，或pub端没有消息产生，sub端的连接都会被强制回收。

```go
func Subs() {
	conn, err := redis.Dial("tcp", "127.0.0.1:6379")
	if err != nil {
		fmt.Println("Connect redis error: ", err)
	}
	defer conn.Close()

	psc := redis.PubSubConn{conn}
	psc.Subscribe("channel1")
	for {
		switch v := psc.Receive().(type) {
		case redis.Message:
			fmt.Printf("%s: message: %s\n", v.Channel, v.Data)
		case redis.Subscription:
			fmt.Printf("%s: %s %d\n", v.Channel, v.Kind, v.Count)
		case error:
			fmt.Println(v)
			return
		}
	}
}

func Push(message string) {
	conn, _ := redis.Dial("tcp", "10.1.210.69:6379")
	_, err := conn.Do("PUBLISH", "channel1", message)
	if err != nil {
		fmt.Println("pub err: ", err)
	}
}
```

### 20.Go 语言相关

#### a.向channel发送数据

##### 向channel发送数据的过程

![channel-direct-send](https://img.draveness.me/2020-01-29-15802354027250-channel-direct-send.png)

发送数据时会调用 [`runtime.send`](https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L270)，该函数的执行可以分成两个部分：

1. 调用 [`runtime.sendDirect`](https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L313) 函数将发送的数据直接拷贝到 `x = <-c` 表达式中变量 `x` 所在的内存地址上；
2. 调用 [`runtime.goready`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L313) 将等待接收数据的 Goroutine 标记成可运行状态 `Grunnable` 并把该 Goroutine 放到发送方所在的处理器的 `runnext` 上等待执行，该处理器在下一次调度时就会立刻唤醒数据的接收方；

##### 向带有缓冲区的cahnnel发送数据的过程

在这里我们首先会使用 `chanbuf` 计算出下一个可以存储数据的位置，然后通过 [`runtime.typedmemmove`](https://github.com/golang/go/blob/db16de920370892b0241d3fa0617dddff2417a4d/src/runtime/mbarrier.go#L156) 将发送的数据拷贝到缓冲区中并增加 `sendx` 索引和 `qcount` 计数器。

![channel-buffer-send](https://img.draveness.me/2020-01-28-15802171487104-channel-buffer-send.png)

`sendx`底层是一个循环数据，因此当`sendx`等于`dataqsiz`时就会重新回到数据开始的位置。

#### b.从channel接收数据

![channel-receive-node](https://img.draveness.me/2020-01-28-15802171487111-channel-receive-node.png)

当 Channel 的 `sendq` 队列中包含处于等待状态的 Goroutine 时，该函数会取出队列头等待的 Goroutine，并调用[`runtime.recv`](https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L556) 函数。

该函数会根据缓冲区的大小分别处理不同的情况：

- 如果 Channel 不存在缓冲区；
  1. 调用 [`runtime.recvDirect`](https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L326) 函数会将 Channel 发送队列中 Goroutine 存储的 `elem` 数据拷贝到目标内存地址中；
- 如果 Channel 存在缓冲区；
  1. 将队列中的数据拷贝到接收方的内存地址；
  2. 将发送队列头的数据拷贝到缓冲区中，释放一个阻塞的发送方；

![channel-receive-from-sendq](https://img.draveness.me/2020-01-28-15802171487118-channel-receive-from-sendq.png)

上图展示了 Channel 在缓冲区已经没有空间并且发送队列中存在等待的 Goroutine 时，运行 `<-ch` 的执行过程 — 发送队列头的 [`runtime.sudog`](https://github.com/golang/go/blob/895b7c85addfffe19b66d8ca71c31799d6e55990/src/runtime/runtime2.go#L342) 结构中的元素会替换接收索引 `recvx` 所在位置的元素，原有的元素会被拷贝到接收数据的变量的内存空间上。

![channel-buffer-receive](https://img.draveness.me/2020-01-28-15802171487125-channel-buffer-receive.png)

如果接收数据的内存地址不为空，那么就会直接使用 [`runtime.typedmemmove`](https://github.com/golang/go/blob/db16de920370892b0241d3fa0617dddff2417a4d/src/runtime/mbarrier.go#L156) 将缓冲区中的数据拷贝到内存中、清除队列中的数据并完成收尾工作。

#### c.context

![golang-context-hierarchy](https://img.draveness.me/golang-context-hierarchy.png)

从源代码来看，[`context.Background`](https://github.com/golang/go/blob/df2999ef43ea49ce1578137017949c0ee660608a/src/context/context.go#L208-L210) 和 [`context.TODO`](https://github.com/golang/go/blob/df2999ef43ea49ce1578137017949c0ee660608a/src/context/context.go#L216-L218) 函数其实也只是互为别名，没有太大的差别。它们只是在使用和语义上稍有不同：

- [`context.Background`](https://github.com/golang/go/blob/df2999ef43ea49ce1578137017949c0ee660608a/src/context/context.go#L208-L210) 是上下文的默认值，所有其他的上下文都应该从它衍生（Derived）出来；
- [`context.TODO`](https://github.com/golang/go/blob/df2999ef43ea49ce1578137017949c0ee660608a/src/context/context.go#L216-L218) 应该只在不确定应该使用哪种上下文时使用；

![golang-parent-cancel-context](https://img.draveness.me/2020-01-20-15795072700927-golang-parent-cancel-context.png)

具体执行流程如下：

1. 当 `parent.Done() == nil`，也就是 `parent` 不会触发取消事件时，当前函数会直接返回；
2. 当`child`的继承链包含可以取消的上下文时，会判断`parent`是否已经触发了取消信号；
   1. 如果已经被取消，`child` 会立刻被取消；
   2. 如果没有被取消，`child` 会被加入 `parent` 的 `children` 列表中，等待 `parent` 释放取消信号；
3. 在默认情况下
   1. 运行一个新的 Goroutine 同时监听 `parent.Done()` 和 `child.Done()` 两个 Channel
   2. 在 `parent.Done()` 关闭时调用 `child.cancel` 取消子上下文；

Go 语言中的 [`context.Context`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/context/context.go#L62-L154) 的主要作用还是在多个 Goroutine 组成的树中同步取消信号以减少对资源的消耗和占用，虽然它也有传值的功能，但是这个功能我们还是很少用到。

#### d.内存分配器

![mutator-allocator-collector](https://img.draveness.me/2020-02-29-15829868066411-mutator-allocator-collector.png)

##### 线性分配器

只需要在内存中维护一个指向内存特定位置的指针，当用户程序申请内存时，分配器只需要检查剩余的空闲内存、返回分配的内存区域并修改指针在内存中的位置。

![bump-allocator](https://img.draveness.me/2020-02-29-15829868066435-bump-allocator.png)

但是线性分配器无法在内存被释放时重用内存。

##### 空闲链表分配器

空闲链表分配器（Free-List Allocator）可以重用已经被释放的内存，它在内部会维护一个类似链表的数据结构。当用户程序申请内存时，空闲链表分配器会依次遍历空闲的内存块，找到足够大的内存，然后申请新的资源并修改链表。

![free-list-allocator](https://img.draveness.me/2020-02-29-15829868066446-free-list-allocator.png)

空闲链表分配器可以选择不同的策略在链表中的内存块中进行选择，最常见的就是以下四种方式：

- 首次适应（First-Fit）— 从链表头开始遍历，选择第一个大小大于申请内存的内存块；
- 循环首次适应（Next-Fit）— 从上次遍历的结束位置开始遍历，选择第一个大小大于申请内存的内存块；
- 最优适应（Best-Fit）— 从链表头遍历整个链表，选择最合适的内存块；
- 隔离适应（Segregated-Fit）— 将内存分割成多个链表，每个链表中的内存块大小相同，申请内存时先找到满足条件的链表，再从链表中选择合适的内存块；

Go 语言使用的内存分配策略与第四种策略有些相似。

![segregated-list](https://img.draveness.me/2020-02-29-15829868066452-segregated-list.png)

##### 分级分配

Go 语言的内存分配器会根据申请分配的内存大小选择不同的处理逻辑，运行时根据对象的大小将对象分成微对象、小对象和大对象三种：

|  类别  |     大小      |
| :----: | :-----------: |
| 微对象 |  `(0, 16B)`   |
| 小对象 | `[16B, 32KB]` |
| 大对象 | `(32KB, +∞)`  |

##### 多机缓存

![multi-level-cache](https://img.draveness.me/2020-02-29-15829868066457-multi-level-cache.png)

当线程缓存不能满足需求时，就会使用中心缓存作为补充解决小对象的内存分配问题；在遇到 32KB 以上的对象时，内存分配器就会选择页堆直接分配大量的内存。

##### 稀疏内存

从Go语言1.11开始，就不再使用连续内存的方式进行分配了，不仅能移除堆大小的上限，还能解决 C 和 Go 混合使用时的地址空间冲突问题。

![heap-after-go-1-11](https://img.draveness.me/2020-02-29-15829868066468-heap-after-go-1-11.png)

运行时使用二维的 [`runtime.heapArena`](https://github.com/golang/go/blob/e7f9e17b7927cad7a93c5785e864799e8d9b4381/src/runtime/mheap.go#L217) 数组管理所有的内存，每个单元都会管理 64MB 的内存空间。

##### 地址空间

|    状态    |                             解释                             |
| :--------: | :----------------------------------------------------------: |
|   `None`   |         内存没有被保留或者映射，是地址空间的默认状态         |
| `Reserved` |        运行时持有该地址空间，但是访问该内存会导致错误        |
| `Prepared` | 内存被保留，一般没有对应的物理内存访问该片内存的行为是未定义的可以快速转换到 `Ready` 状态 |
|  `Ready`   |                        可以被安全访问                        |

![memory-regions-states-and-transitions](https://img.draveness.me/2020-02-29-15829868066474-memory-regions-states-and-transitions.png)

#### e.内存管理组件

Go 语言的内存分配器包含内存管理单元、线程缓存、中心缓存和页堆几个重要组件，本节将介绍这几种最重要组件对应的数据结构 [`runtime.mspan`](https://github.com/golang/go/blob/921ceadd2997f2c0267455e13f909df044234805/src/runtime/mheap.go#L358)、[`runtime.mcache`](https://github.com/golang/go/blob/01d137262a713b308c4308ed5b26636895e68d89/src/runtime/mcache.go#L19)、[`runtime.mcentral`](https://github.com/golang/go/blob/8ac98e7b3fcadc497c4ca7d8637ba9578e8159be/src/runtime/mcentral.go#L20) 和 [`runtime.mheap`](https://github.com/golang/go/blob/8ac98e7b3fcadc497c4ca7d8637ba9578e8159be/src/runtime/mheap.go#L40)。

![go-memory-layout](https://img.draveness.me/2020-02-29-15829868066479-go-memory-layout.png)

##### 内存管理单元

[`runtime.mspan`](https://github.com/golang/go/blob/921ceadd2997f2c0267455e13f909df044234805/src/runtime/mheap.go#L358) 是 Go 语言内存管理的基本单元，该结构体中包含 `next` 和 `prev` 两个字段。运行时会使用 [`runtime.mSpanList`](https://github.com/golang/go/blob/921ceadd2997f2c0267455e13f909df044234805/src/runtime/mheap.go#L352) 存储双向链表的头结点和尾节点并在线程缓存以及中心缓存中使用。每个 [`runtime.mspan`](https://github.com/golang/go/blob/921ceadd2997f2c0267455e13f909df044234805/src/runtime/mheap.go#L358) 都管理 `npages` 个大小为 8KB 的页，这里的页不是操作系统中的内存页，它们是操作系统内存页的整数倍。

```go
type mspan struct {
    next *mspan
    prev *mspan
    ...
}
```

![mspan-and-linked-list](https://img.draveness.me/2020-02-29-15829868066485-mspan-and-linked-list.png)

##### 线程缓存

[`runtime.mcache`](https://github.com/golang/go/blob/01d137262a713b308c4308ed5b26636895e68d89/src/runtime/mcache.go#L19) 是 Go 语言中的线程缓存，它会与线程上的处理器一一绑定，主要用来缓存用户程序申请的微小对象。每一个线程缓存都持有 67 * 2 个 [`runtime.mspan`](https://github.com/golang/go/blob/921ceadd2997f2c0267455e13f909df044234805/src/runtime/mheap.go#L358)，这些内存管理单元都存储在结构体的 `alloc` 字段中。

![mcache-and-mspans](https://img.draveness.me/2020-02-29-15829868066512-mcache-and-mspans.png)

##### 中心缓存

[`runtime.mcentral`](https://github.com/golang/go/blob/8ac98e7b3fcadc497c4ca7d8637ba9578e8159be/src/runtime/mcentral.go#L20) 是内存分配器的中心缓存，与线程缓存不同，访问中心缓存中的内存管理单元需要使用互斥锁。

```go
type mcentral struct {
	lock      mutex
	spanclass spanClass
	nonempty  mSpanList
	empty     mSpanList
	nmalloc uint64
}
```

它同时持有两个[`runtime.mSpanList`]()，分别存储包含空闲对象的列表和不包含空闲对象的链表。

![mcentral-and-mspans](https://img.draveness.me/2020-02-29-15829868066519-mcentral-and-mspans.png)

线程缓存会通过中心缓存的 [`runtime.mcentral.cacheSpan`](https://github.com/golang/go/blob/8ac98e7b3fcadc497c4ca7d8637ba9578e8159be/src/runtime/mcentral.go#L40) 方法获取新的内存管理单元，该方法的实现比较复杂，我们可以将其分成以下几个部分：

1. 从有空闲对象的 [`runtime.mspan`](https://github.com/golang/go/blob/921ceadd2997f2c0267455e13f909df044234805/src/runtime/mheap.go#L358) 链表中查找可以使用的内存管理单元；
2. 从没有空闲对象的 [`runtime.mspan`](https://github.com/golang/go/blob/921ceadd2997f2c0267455e13f909df044234805/src/runtime/mheap.go#L358) 链表中查找可以使用的内存管理单元；
3. 调用 [`runtime.mcentral.grow`](https://github.com/golang/go/blob/8ac98e7b3fcadc497c4ca7d8637ba9578e8159be/src/runtime/mcentral.go#L251) 从堆中申请新的内存管理单元；
4. 更新内存管理单元的 `allocCache` 等字段帮助快速分配内存；

##### 页堆

[`runtime.mheap`](https://github.com/golang/go/blob/8ac98e7b3fcadc497c4ca7d8637ba9578e8159be/src/runtime/mheap.go#L40) 是内存分配的核心结构体，Go 语言程序只会存在一个全局的结构，而堆上初始化的所有对象都由该结构体统一管理，该结构体中包含两组非常重要的字段，其中一个是全局的中心缓存列表 `central`，另一个是管理堆区内存区域的 `arenas` 以及相关字段。

页堆中包含一个长度为 134 的 [`runtime.mcentral`](https://github.com/golang/go/blob/8ac98e7b3fcadc497c4ca7d8637ba9578e8159be/src/runtime/mcentral.go#L20) 数组，其中 67 个为跨度类需要 `scan` 的中心缓存，另外的 67 个是 `noscan` 的中心缓存。

![mheap-and-mcentrals](https://img.draveness.me/2020-02-29-15829868066525-mheap-and-mcentrals.png)

堆区的初始化会使用 [`runtime.mheap.init`](https://github.com/golang/go/blob/8ac98e7b3fcadc497c4ca7d8637ba9578e8159be/src/runtime/mheap.go#L672) 方法，其中初始化的两类变量比较重要。

1. `spanalloc`、`cachealloc` 以及 `arenaHintAlloc` 等 [`runtime.fixalloc`](https://github.com/golang/go/blob/44dcb5cb61aee5435e0b3c78544a1d3352a4cc98/src/runtime/mfixalloc.go#L27) 类型的空闲链表分配器；
2. `central` 切片中 [`runtime.mcentral`](https://github.com/golang/go/blob/8ac98e7b3fcadc497c4ca7d8637ba9578e8159be/src/runtime/mcentral.go#L20) 类型的中心缓存；

[`runtime.mheap.grow`](https://github.com/golang/go/blob/6dd11bcb35cba37f5994c1b9aaaf7d2dc13fd7cf/src/runtime/mheap.go#L1276) 方法会向操作系统申请更多的内存空间。在页堆扩容的过程中，[`runtime.mheap.sysAlloc`](https://github.com/golang/go/blob/6dd11bcb35cba37f5994c1b9aaaf7d2dc13fd7cf/src/runtime/malloc.go#L617) 是页堆用来申请虚拟内存的方法。

#### f.内存分配

堆上所有的对象都会通过调用 [`runtime.newobject`](https://github.com/golang/go/blob/921ceadd2997f2c0267455e13f909df044234805/src/runtime/malloc.go#L1164) 函数分配内存，该函数会调用 [`runtime.mallocgc`](https://github.com/golang/go/blob/921ceadd2997f2c0267455e13f909df044234805/src/runtime/malloc.go#L891) 分配指定大小的内存空间，这也是用户程序向堆上申请内存空间的必经函数。

##### 微对象

使用线程缓存上的微分配器提高微对象分配的性能，主要使用它来分配较小的字符串以及逃逸的临时变量。微分配器可以将多个较小的内存分配请求合入同一个内存块中，只有当内存块中的所有对象都需要被回收时，整片内存才可能被回收。

##### 小对象

小对象的分配可以被分成以下的三个步骤：

1. 确定分配对象的大小以及跨度类 [`runtime.spanClass`](https://github.com/golang/go/blob/921ceadd2997f2c0267455e13f909df044234805/src/runtime/mheap.go#L503)；
2. 从线程缓存、中心缓存或者堆中获取内存管理单元并从内存管理单元找到空闲的内存空间；
3. 调用 [`runtime.memclrNoHeapPointers`](https://github.com/golang/go/blob/05c02444eb2d8b8d3ecd949c4308d8e2323ae087/src/runtime/memclr_386.s#L13) 清空空闲内存中的所有数据；

##### 大对象

运行时对于大于 32KB 的大对象会单独处理，不会从线程缓存或者中心缓存中获取内存管理单元，而是直接在系统的栈中调用 [`runtime.largeAlloc`](https://github.com/golang/go/blob/6dd11bcb35cba37f5994c1b9aaaf7d2dc13fd7cf/src/runtime/malloc.go#L1136) 函数分配大片的内存。

#### g.切片

##### 数据结构

```go
type SliceHeader struct {
    Data uintptr
    Len  int
    Cap  int
}
```

##### 扩容规则

Go 语言根据切片的当前容量选择不同的策略进行扩容：

1. 如果期望容量大于当前容量的两倍就会使用期望容量；
2. 如果当前切片的长度小于 1024 就会将容量翻倍；
3. 如果当前切片的长度大于 1024 就会每次增加 25% 的容量，直到新容量大于期望容量；

#### h.哈希表

##### 哈希冲突处理

[开放寻址法](https://en.wikipedia.org/wiki/Open_addressing)是一种在哈希表中解决哈希碰撞的方法，这种方法的核心思想是**对数组中的元素依次探测和比较以判断目标键值对是否存在于哈希表中**，如果我们使用开放寻址法来实现哈希表，那么在支撑哈希表的数据结构就是数组。

發生衝突時，就会将键值对写入到下一个不为空的位置：

![open-addressing-and-set](https://img.draveness.me/2019-12-30-15777168478785-open-addressing-and-set.png)

开放寻址法中对性能影响最大的就是**装载因子**，它是数组中元素的数量与数组大小的比值，随着装载因子的增加，线性探测的平均用时就会逐渐增加，这会同时影响哈希表的读写性能。

与开放地址法相比，拉链法是哈希表中最常见的实现方法，大多数的编程语言都用拉链法实现哈希表，它的实现比较开放地址法稍微复杂一些，但是平均查找的长度也比较短，各个用于存储节点的内存都是动态申请的，可以节省比较多的存储空间。

![separate-chaing-and-set](https://img.draveness.me/2019-12-30-15777168478798-separate-chaing-and-set.png)

##### 数据结构

```go
type hmap struct {
	count     int       // 表示当前哈希表中的元素数量
	flags     uint8
	B         uint8     // 表示当前哈希表持有的 buckets 数量
	noverflow uint16
	hash0     uint32    // 哈希的种子

	buckets    unsafe.Pointer
	oldbuckets unsafe.Pointer   // 哈希在扩容时用于保存之前 buckets 的字段
	nevacuate  uintptr

	extra *mapextra
}
```

![hmap-and-buckets](https://img.draveness.me/2019-12-30-15777168478811-hmap-and-buckets.png)

##### 遍历

在 `bucketloop` 循环中，哈希会依次遍历正常桶和溢出桶中的数据，它会比较这 8 位数字和桶中存储的 `tophash`，每一个桶都存储键对应的 `tophash`，每一次读写操作都会与桶中所有的 `tophash` 进行比较，用于选择桶序号的是哈希的最低几位，而用于加速访问的是哈希的高 8 位，这种设计能够减少同一个桶中有大量相等 `tophash` 的概率。

![hashtable-mapaccess](https://img.draveness.me/2019-12-30-15777168478817-hashtable-mapaccess.png)

##### 扩容

[`runtime.mapassign`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L571-L683) 函数会在以下两种情况发生时触发哈希的扩容：

1. 装载因子已经超过 6.5；
2. 哈希使用了太多溢出桶；

不过由于 Go 语言哈希的扩容不是一个原子的过程，所以 [`runtime.mapassign`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L571-L683) 函数还需要判断当前哈希是否已经处于扩容状态，避免二次扩容造成混乱。

![hashtable-hashgrow](https://img.draveness.me/2019-12-30-15777168478830-hashtable-hashgrow.png)