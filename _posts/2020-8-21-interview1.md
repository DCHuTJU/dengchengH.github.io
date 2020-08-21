---
layout: post
title: 【面试】面经总结（一）
tags: interview
stickie: true
---

### 1.LRU算法

`LRU(Least Recently Used)`缓存算法，最近最久未使用。常用于页面置换算法。是为虚拟页式存储管理服务的。对于在内存中又不用的数据块，称为`LRU`，操作系统会根据哪些数据属于`LRU`而将其移出内存。实现的基本方法有以下几种。

#### a.单链表实现

* `put`操作

1. 如果要`put`的元素已经在链表中，需要在原来的位置把旧的数据删除，然后在链表的头部插入新的数据。
2. 如果要`put`的元素不在链表中，则需要先判断缓存区是否已满，如果满的话，则把链表尾部的节点删除，之后把新的数据插入到链表头部。如果没有满的话，则直接把数据插入链表头部即可。

* `get`操作

1. 如果要`get(key)` 的数据存在于链表中，则把 `value` 返回，并且把该节点删除，删除之后把它插入到链表的头部。
2. 如果要`get(key)`的数据不存在于链表中，则直接返回`-1`即可。

>时间复杂度为`O(n)`，空间复杂度为`O(1)`

#### b.空间换时间

在实际的应用中，`put` 操作一般伴随着` get` 操作，也就是说，`get` 操作的次数是比较多的，而且命中率也是相对比较高的，而 `put` 操作的次数是比较少的，可以考虑采用**空间换时间**的方式来加快`get` 的操作。

要想将`get`操作控制在`O(1)`的时间复杂度，可以使用`Hash`存放`key-value`（Go中的`map`）。用了哈希表之后，虽然我们能够在` O(1)` 时间内找到目标元素，可以，我们还需要删除该元素，并且把该元素插入到链表头部啊，删除一个元素，我们是需要定位到这个元素的**前驱**的，然后定位到这个元素的前驱，是需要 `O(n)` 时间复杂度的。

#### c.双向链表+哈希表

在删除时，需要额外的指针指在被删除节点的前驱上，此时可以使用**双链表**进行替换。

```go
// 双链表可以使用 list.List 实现
type LRU struct {
    maxLength int // 最大长度
    usedLength int // 已经使用长度
    ll *list.List  // 双链表
    cache map[string]*list.Element // 用来实现 O(1) 的查询
}
// 内容实体
type Entry struct {
    key string
    value interface{}
}

func (e *entry) Len() int {
	return cache.CalcLen(e.value)
}
// New 创建一个新的 Cache，如果 maxBytes 是 0，表示没有容量限制
func New(maxBytes int, onEvicted func(key string, value interface{})) cache.Cache {
	return &lru{
		maxBytes:  maxBytes,
		onEvicted: onEvicted,
		ll:        list.New(),
		cache:     make(map[string]*list.Element),
	}
}
// Put 往 Cache 尾部增加一个元素（如果已经存在，则放入尾部，并更新值）
func (l *LRU) Put(key string, value interface{}) {
    if e, ok := l.cache[key]; ok {
        l.ll.MoveToBack(e)
        en := e.Value.(*Entry)
        l.usedLength = l.usedLength - cache.CalcLen(en.value) + cache.CalcLen(value)
        en.value = value
        return
    }
    
    en := &entry{key, value}
    e := l.ll.PushBack(en)
    l.cache[key] = e
    
    l.usedLength += en.Len()
    if l.maxLength > 0 && l.usedLength > l.maxLength {
        l.DelOldest()
    }
}
// Get 从 cache 中获取 key 对应的值，nil 表示 key 不存在
func (l *LRU) Get(key string) interface{} {
    if e, ok := l.cache[key]; ok {
        l.ll.MoveToBack(e)
        return e.Value.(*Entry).value
    }
    return nil
}
// Del 从 cache 中删除 key 对应的元素
func (l *lru) Del(key string) {
	if e, ok := l.cache[key]; ok {
		l.removeElement(e)
	}
}
// DelOldest 从 cache 中删除最旧的记录
func (l *lru) DelOldest() {
	l.removeElement(l.ll.Front())
}
// Len 返回当前 cache 中的记录数
func (l *lru) Len() int {
	return l.ll.Len()
}

func (l *LRU) removeElement(e *list.Element) {
    if e == nil {
		return
	}
    
    l.ll.Remove(e)
    en := e.Value.(*Entry)
    l.usedLength -= en.Len()
    delete(l.cache, en.key)
    
    if l.onEvicted != nil {
        l.onEvicted(en.key, en.value)
    }
}
```

### 2.一致性Hash

#### a.为什么不能直接使用Hash

直接使用取模`Hash`算法时，会出现一些缺陷，主要体现在服务器数量变动的时候，所有缓存的位置都要发生改变。当服务器数量发生改变时，所有缓存在一定时间内是失效的，当应用无法从缓存中获取数据时，则会向后端数据库请求数据（导致**缓存雪崩**）。

#### b.一致性Hash

一致性Hash算法也是使用取模的方法，只不过一致性`Hash`算法是对$2^{32}$取模。整个空间按照**顺时针方向组织**。将由这$2^{32}$个点组成的圆环称为**Hash环**。

<img src="https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_jpg/UtWdDgynLdbhiae1AfNYAibdp7ib2wTZTrpb4iaIwTkwrloXHV4ebuemEQsqkhTveAlFCN9TQA4YN5mNbAO1tutyow/640" alt="一致性Hash" style="zoom:50%;" />

接下来使用如下算法定位数据访问到相应服务器：将数据`key`使用相同的函数`Hash`计算出哈希值，并确定此数据在环上的位置，从此位置沿环顺时针“行走”，第一台遇到的服务器就是其应该定位到的服务器。

#### c.一致性Hash的容错性和可扩展性

在一致性Hash算法中，如果一台服务器不可用，则受影响的数据仅仅是此服务器到其环空间中前一台服务器（即沿着逆时针方向行走遇到的第一台服务器）之间数据，其它不会受到影响。

<img src="https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_jpg/UtWdDgynLdbhiae1AfNYAibdp7ib2wTZTrppjapqr7nD1s7BeSGmJKMichyiaQJrIqtatg5WibKY4yYc0bdJjrticu73g/640" alt="一致性Hash的容错性" style="zoom:50%;" />

在一致性Hash算法中，如果增加一台服务器，则受影响的数据仅仅是新服务器到其环空间中前一台服务器（即沿着逆时针方向行走遇到的第一台服务器）之间数据，其它数据也不会受到影响。

<img src="https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_jpg/UtWdDgynLdbhiae1AfNYAibdp7ib2wTZTrpA5jBRhZxgl6Uh78tMibBKebtnwIQsUibPRV0XExnhTnmgvNbBJSmFicOA/640" alt="一致性Hash扩展性" style="zoom:50%;" />

#### d.Hash环的数据倾斜问题

一致性Hash算法在**服务节点太少时**，容易因为节点分部不均匀而造成**数据倾斜**（被缓存的对象大部分集中缓存在某一台服务器上）问题。为了解决这种数据倾斜问题，一致性Hash算法引入了**虚拟节点机制**，即对每一个服务节点计算多个哈希，每个计算结果位置都放置一个此服务节点，称为**虚拟节点**。具体做法可以在服务器IP或主机名的后面增加编号来实现。

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
    idx := sort.Search(len(m.keys), func(i int) bool { return m.keys[i] >= hash })
    // 如果查找结果大于节点哈希数组的最大索引，表示此时该对象哈希值位于最后一个节点之后，那么放入第一个节点中
    if idx == len(m.keys) {
        idx = 0
    }

    return m.hashMap[m.keys[idx]]
}
```

### 3.I/O复用

#### a.Unix中的五种I/O模型

* 阻塞式I/O

<img src="https://user-gold-cdn.xitu.io/2019/3/31/169d16a9a67b8a0a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1" alt="img" style="zoom:50%;" />

具体过程就是首先应用进程发起系统调用，会进入等待，一直等到数据报就绪，这时候数据还是在内核缓冲区中，需要将数据报返回给应用进程缓冲区。

> 阻塞式 I/O 不是意味着系统进入阻塞，而仅仅是当前应用程序阻塞，其他应用程序还是可以继续运行的，因此不消耗 CPU 时间，执行效率较高

* 非阻塞式I/O

<img src="https://user-gold-cdn.xitu.io/2019/3/31/169d18e8f8033ded?imageView2/0/w/1280/h/960/format/webp/ignore-error/1" alt="img" style="zoom:50%;" />

具体过程是采用轮询的方式不断进行系统调用来获取I/O是否完成的结果。

* I/O复用

<img src="https://user-gold-cdn.xitu.io/2019/3/31/169d19896b6c3b37?imageView2/0/w/1280/h/960/format/webp/ignore-error/1" alt="img" style="zoom:50%;" />

具体过程主要是先调用`select`，系统监听所有`select`负责的数据报，一旦有某个数据准备就绪，就会将其返回。然后进行`recvfrom`系统调用，执行同阻塞式`I/O`相同的处理。

> 由于I/O复用需要调用两个系统调用，所以效率肯定不如前者，但是最大的特点就是可以同时处理多个 connection。

* 异步I/O

<img src="https://user-gold-cdn.xitu.io/2019/3/31/169d1a09ec4b4e01?imageView2/0/w/1280/h/960/format/webp/ignore-error/1" alt="img" style="zoom: 50%;" />

进行 `aio_read` 系统调用会立即返回， 应用进程继续执行， 不会被阻塞， 内核会在所有操作完成之后向应用进程发送信号。

![io](https://ninokop.github.io/2018/02/18/go-net/io.png)

#### b.阻塞与非阻塞的区别

阻塞式和非阻塞式最大的区别就是阻塞式在等待数据阶段会进入阻塞，而非阻塞式不会，但是对于非阻塞式，在获得数据存在内核缓冲区后，将内核缓冲区中数据复制到应用程序缓冲区这个阶段是阻塞的。

#### c.同步与异步的区别

同步与异步的区别在于在进行 I/O 操作时候会将进程阻塞，根据这个定义就知道，阻塞式、非阻塞式、信号驱动式、I/O 复用式都属于同步。

#### d.I/O多路复用

* select

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
3. 能够从通知中得到有可读事件的fds列表，而不是需要遍历整个fds来收集

* poll

`poll`最大的特点在于没有最大文件描述数量的限制。poll改变了fds集合的描述方式，使用了pollfd结构而不是select的fd_set结构，使得poll支持的fds集合限制远大于select的1024，其中 pollfd 使用链表实现。

```cpp
int poll(struct pollfd *fds, unsigned int nfds, int timeout);
```

* select 与 poll 的区别

> 1.功能
>
> * `select` 的描述符类型使用数组实现，`FD_SETSIZE` 默认大小为 `1024`，不过这个值可以改变，如果需要修改的话要重新编译；而 `poll` 使用链表实现，没有描述符大小的限制；
>
> 2.速度
>
> * 速度都很慢；
> * 共同的就是在调用时都需要将全部描述符从应用进程缓冲区复制到内核缓冲区；
>
> * 两者返回结果中没有声明哪些描述符已经准备好，所以如果返回值大于 0 时，应用进程都需要使用轮询的方式来找到 I/O 完成的描述符；

* epoll

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

![img](https://raw.githubusercontent.com/panjf2000/illustrations/master/go/epoll-principle.png?imageView2/2/w/1280/format/jpg/interlace/1/q/100)

* epoll 工作模式

> 1. LT 模式
>
>    当 `epoll_wait()` 检测到描述符事件到达时，将此时间通知进程，进程**可以**不立即处理该事件，下次调用 `epoll_wait()` 时会再次通知进程，这是默认一种模式，并且同时支持阻塞和非阻塞.
>
> 2. ET 模式
>
>    和 LT 模式不同的是，通知之后必须立即处理事件，下次再调用 epoll_wait() 不会再得到时间到达的通知。
>
>    减少了 epoll 事件被重复触发的次数，因此效率比 LT 高，只支持非阻塞式，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。

#### e.多路复用应用场景

* select 使用场景
  * select 的 timeout 精度为 1ns，而其他两种为 1ms，所以 select 更适用于实时要求很高的场景，比如核反应堆的控制
* poll 使用场景
  * `poll` 与 `select` 相比没有最大描述符数量的限制，并且如果平台对实时性要求不是很高，一般使用`poll`
  * 需要同时监控小于 1000 个描述符，就没必要使用 `epoll`，因为这个应用场景下并不能体现 `epoll` 的优势

* epoll 使用场景
  * 有非常大量的描述符需要同时轮询，而且这些连接最好是长连接

#### f.服务器常用编程模型

* 多线程
* IO多路复用

实际上目前的高性能服务器很多都用的是`reactor`模式，即`non-blocking IO`+`IO multiplexing`的方式。通常主线程只做`event-loop`，通过`epoll_wait`等方式监听事件，而处理客户请求是在其他工作线程中完成。

> 传统的网络实现
>
> 1. `server`端在`bind&listen`后，将`listenfd`注册到`epollfd`中，最后进入`epoll_wait`循环。循环过程中若有在`listenfd`上的`event`则调用`socket.accept`，并将`connfd`加入到`epollfd`的IO复用队列。
> 2. 当`connfd`上有数据到来或写缓冲有数据可以触发`epoll_wait`的返回，这里读写IO都是非阻塞IO，这样才不会阻塞`epoll`的下一个循环。然而，这样容易割裂业务逻辑，不易理解和维护。
> 3. `read`后数据进行解码并放入队列中，等待工作线程处理。

![io](https://ninokop.github.io/2018/02/18/go-net/c-net.jpg)

#### g.Go中实现netPoll

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

* netFD

在Go中，服务端通过listen建立起的`Listener`是个实现了**Accept Close**等方法的接口。通过`listener`的`Accept`方法返回的Conn是一个实现了**Read Write**等方法的接口。`Listener`和`Conn`的实现都包含一个网络文件描述符`netFD`，它其中包含一个重要的数据结构`pollDesc`，它是底层事件驱动的封装。

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

1. 服务端的`netFD`在`listen`时会创建`epoll`的实例，并将`listenFD`加入`epoll`的事件队列
2. `netFD`在`accept`时将返回的`connFD`也加入`epoll`的事件队列
3. `netFD`在读写时出现`syscall.EAGAIN`错误，通过`pollDesc`将当前的`goroutine` `park`住，直到`ready`，从`pollDesc`的`waitRead`中返回

* pollDesc

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

* netpoll_epoll

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

它封装成了四个`runtime`函数。**netpollinit** 使用`epoll_create`创建`epollfd`，**netpollopen** 添加一个`fd`到`epoll`中，这里的数据结构称为`pollDesc`，它一开始都关注了**读写事件**，并且采用的是**边缘触发**。**netpollclose**函数就是从`epoll`删除一个`fd`。**netpoll** 就是从`epoll wait`得到所有发生事件的`fd`，并将每个`fd`对应的`goroutine`通过链表返回。

### 4.线程池（协程池）

#### a.什么是线程池

线程池是一种多线程处理形式，处理过程中将任务添加到队列，然后在创建线程后自动启动这些任务。

#### b.为什么使用线程池

* 降低资源消耗
  * 通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
* 提高响应速度
  * 当任务到达时，任务可以不需要等到线程创建就能立即执行。
* 提高线程的可管理性
  * 线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。

#### b.最简单的方法

思路其实非常简单，用一个`channel`当做任务队列，初始化`groutine`池时确定好并发量，然后以设置好的并发量开启`groutine`同时读取`channel`中的任务并执行。

```go
type SimplePool struct {
    wg sync.WaitGroup
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

#### b.进阶版

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

### 5.HTTP与HTTPS

![image-20200819215931938](https://s1.ax1x.com/2020/08/21/dYaVNn.png)

### 6.数据库范式

#### a.第一范式

每一列属性都是不可再分的属性值，确保每一列的原子性。

#### b.第二范式

在满足第一范式的基础上，实体的每个非主键属性完全函数依赖于主键属性（消除部分依赖）。

>第二范式产生的影响
>
>* 数据信息冗余
>* 增删改会出现问题

解决办法：使用关系分解方法消除部分依赖

#### c.第三范式

在满足第二范式的基础上，在实体中不存在非主键属性传递函数依赖于主键属性。（表中字段[非主键]不存在对主键的传递依赖）

> 第三范式产生的影响
>
> * 数据信息冗余
> * 更新时需要同时修改多条记录，否则会出现数据不一致的情况

解决办法：使用关系分解方法消除部分依赖

#### d.什么时候需要突破范式

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

### 7.索引创建与优化

#### a.分析原因

1. 先观察，开启慢查询日志，设置相应的阈值（比如超过3秒就是慢`SQL`），在生产环境跑上个一天过后，看看哪些`SQL`比较慢
2. `Explain`和慢`SQL`分析
3. `Show Profile`是比`Explain`更近一步的执行细节，可以查询到执行每一个`SQL`都干了什么事，这些事分别花了多少秒

#### b.Explain 详解

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

#### c.是否需要创建索引

| 需要创建索引                                                 | 不需要创建索引                                               |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 主键自动建立唯一索引                                         | 表记录过少                                                   |
| 频繁作为查询条件的字段应该创建索引                           | 经常增删改的表或者字段（虽然可以提高查询速度，但是会降低更新表的速度，因为更新表时，不仅要保存数据，还要保存一下索引文件） |
| 查询中与其它表关联的字段，外键关系建立索引                   | `where`条件里用不到的字段不创建索引                          |
| 查询中排序的字段，排序字段若通过索引去访问将大大提高排序速度 | 过滤性不好的不适合建索引，如性别等                           |

#### d.SQL语法顺序&执行顺序

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

#### e.避免不走索引的场景

* 尽量避免在字段开头模糊查询，会导致数据库引擎放弃索引进行全表扫描。

```SQL
SELECT * FROM t WHERE username LIKE '%陈%';
SELECT * FROM t WHERE username LIKE '陈%';
```

如果要求使用模糊查询，则使用`MySQL`内置函数`INSTR(str, substr)`来匹配。数据量较少时，直接用`like '%xx%'`即可。

* 尽量避免使用 in 和 not in，会导致引擎走全表扫描

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

* 尽量避免使用 or，会导致数据库引擎放弃索引进行全表扫描

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

#### f.SELECT 语句优化

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

#### g.增删改DML语句优化

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

### 8.TCP/IP四层结构，每层有哪些协议

![image-20200724095019093](https://s1.ax1x.com/2020/08/21/dYaucT.png)

<img src="https://img-blog.csdn.net/20180930155137505?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NzZG5fa291/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70" alt="TCP/IP四层协议" style="zoom:150%;" />

#### a.数据链路层

* ARP（Address Resolve Protocol，地址解析协议）

  网络层必须先将目标机器的IP地址转化成其物理地址，才能使用数据链路层提供的服务。

* RARP（Reverse Address Resolve Protocol，逆地址解析协议）

  运行RARP服务的网络管理者通常存有该网络上所有机器的物理地址到IP地址的映射。

#### b.网络层

* IP（Internet Protocol，因特网协议）

  IP协议使用逐跳（hop by hop）的方式确定通信路径。

* ICMP（Internet Control Message Protocol，因特网控制报文协议）

  主要用于检测网络连接。

  * 差错报文：用来回应网络错误，比如目标不可达（3）和重定向（5）；
  * 查询报文：查询网络信息，比如ping程序就是使用ICMP报文查看目标是否可到达（8）的。

  ICMP报文使用16位校验和字段对整个报文（包括头部和内容部分）进行循环冗余校验（Cyclic Redundancy Check，CRC），以检验报文在传输过程中是否损坏。不同的ICMP报文类型具有不同的正文内容。

#### c.传输层

* TCP（Transmission Control Protocol，传输控制协议）

  为应用层提供可靠的、面向连接的和基于流（stream）的服务。TCP协议使用超时重传、数据确认等方式来确保数据包被正确地发送至目的端。TCP服务是基于流的。基于流的数据没有边界（长度）限制，它源源不断地从通信的一端流入另一端。发送端可以逐个字节地向数据流中写入数据，接收端也可以逐个字节地将它们读出。

* UDP（User Datagram Protocol，用户数据报协议）

  为应用层提供不可靠、无连接和基于数据报的服务。基于数据报的服务，是相对基于流的服务而言的。每个UDP数据报都有一个长度，接收端必须以该长度为最小单位将其所有内容一次性读出，否则数据将被截断。

### 9.DNS服务器原理

#### a.DNS解析过程

![img](https://pic1.zhimg.com/80/v2-30bdd430e5066476a87f977009f990cf_720w.jpg)

#### b.DNS什么时候使用TCP，什么时候使用UDP

DNS在域名解析过程中使用UDP，在主DNS服务器与辅助DNS服务器之间通信，并加载数据信息（又称为区域复制）过程中使用TCP。

* 域名解析使用UDP

  因为UDP块

* 区域传送使用TCP

  因为TCP可靠

#### c.DNS原理

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

### 10.线程的五种状态及转换

#### a.线程5种状态（JAVA）

* 新建状态
* 就绪状态
* 运行状态
* 阻塞状态
  * 等待阻塞：`wait()`
  * 同步阻塞：请求获取同步锁时，该同步锁被别的线程占用
  * 其他阻塞：`sleep()`或`join()`
* 死亡状态

<img src="http://img.blog.csdn.net/20140828202610671" alt="img" style="zoom:150%;" />

#### b.协程6种状态（Go）

![img](https://static.studygolang.com/171207/80c5272a28b7b5ed5a1bf3f24157dd27.jpg)

* 初始化状态（只是分配的地址，但是并未
* 就绪状态
* 运行状态
* 系统调用状态（保证高的并发性能）
* 阻塞状态
* 死亡状态

#### c.线程与进程之间的资源的关系

一个进程由一到多个线程组成，各线程共享进程的内存空间（代码，数据，堆）和一些进程级的资源（打开的文件和信号）。进程有自己独立的寄存器和栈。

线程私有的是：局部变量，函数的参数，TLS（Thread Local Storage，线程局部存储）数据。

线程之间共享（进程所有）：全局变量，堆，函数里的静态变量，程序代码，打开的文件。

### 11.动态链接库与静态链接库的区别

都属于共享代码的方式。

* 静态链接

动态链接的基本思想是把程序按照模块拆分成各个相对独立部分，在程序运行时才将它们链接在一起形
成一个完整的程序，而不是像静态链接一样把所有程序模块都链接成一个单独的可执行文件。
共享库：就是即使需要每个程序都依赖同一个库，但是该库不会像静态链接那样在内存中存在多分，副
本，而是这多个程序在执行时共享同一份副本；
更新方便：更新时只需要替换原来的目标文件，而无需将所有的程序再重新链接一遍。当程序下一次运
行时，新版本的目标文件会被自动加载到内存并且链接起来，程序就完成了升级的目标。
性能损耗：因为把链接推迟到了程序运行时，所以每次执行程序都需要进行链接，所以性能会有一定损
失。

* 静态链接

函数和数据被编译进一个二进制文件。在使用静态库的情况下，在编译链接可执行文件时，链接器从库
中复制这些函数和数据并把它们和应用程序的其它模块组合起来创建最终的可执行文件。
空间浪费：因为每个可执行程序中对所有需要的目标文件都要有一份副本，所以如果多个程序对同一个
目标文件都有依赖，会出现同一个目标文件都在内存存在多个副本；
更新困难：每当库函数的代码修改了，这个时候就需要重新进行编译链接形成可执行程序。
运行速度快：但是静态链接的优点就是，在可执行程序中已经具备了所有执行程序所需要的任何东西，
在执行的时候运行速度快。

### 12.CPU多级缓存

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

#### a.cache 缓存带来的问题

导致数据不一致的问题，对于多核系统来说。

#### b.缓存一致性

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

### 13.三次握手与四次挥手

#### a.TCP

- TCP 提供一种**面向连接的、可靠的**字节流服务
- 在一个 TCP 连接中，仅有两方进行彼此通信。广播和多播不能用于 TCP
- TCP 使用校验和，确认和重传机制来保证可靠传输
- TCP 给数据分节进行排序，并使用累积确认保证数据的顺序不变和非重复
- TCP 使用滑动窗口机制来实现流量控制，通过动态改变窗口的大小进行拥塞控制

#### b.三次握手

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

#### c.四次挥手

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

### 14.浏览器URL的请求过程

* DNS解析
* TCP连接
* 发送HTTP请求
* 服务器处理请求并返回HTTP报文
* 浏览器解析并渲染页面
* 连接结束

![image-20200821101209174](https://s1.ax1x.com/2020/08/21/dYaZhq.png)

#### a.GET与POST的区别

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

#### b.HTTP状态码

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

#### c.HTTP报文

一个HTTP请求报文由请求行（request line）、请求头部（header）、空行和请求数据4个部分组成。

![img](https://pic002.cnblogs.com/images/2012/426620/2012072810301161.png)

