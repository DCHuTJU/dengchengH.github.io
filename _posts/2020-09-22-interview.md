---
layout: post
title: 【面试】边边角角（一）
tags: interview
---

### 计算机网络

#### 1. 应用层

把应用层交互的数据单元称为报文。

> 域名系统
>
> 域名系统`(Domain Name System, DNS)`是因特网的一项核心服务，它作为可以将域名和`IP`地址相互映射的一个分布式数据库，能够使人更方便的访问互联网，而不用去记住能够被机器直接读取的`IP`地址。
>
> ```shell
> 在Linux中，通常DNS配置文件位于 resolv.conf 文件中
> ```

#### 2. 传输层

#### 粘包问题

![image-20200921092211133](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200921092211133.png)

产生粘包的原因有两种：

1. 当连续发送数据时，由于`tcp`协议的`nagle`算法，会将较小的内容拼接成大的内容，一次性发送到服务器端，因此造成粘包；
2. 当发送内容较大时，由于服务器端的`recv(buffer_size)`方法中的`buffer_size`较小，不能一次性完全接收全部内容，因此在下一次请求到达时，接收的内容依然是上一次没有完全接收完的内容，因此造成粘包。

解决方法：

1. 在两次`send()`直接使用`recv()`来阻止连续发送的情况。
2. 由于产生粘包的原因是接收方的无边界接收，因此发送端可以在发送数据之前向接收端告知发送内容的大小即可。

#### 握手&挥手

##### a. 为什么需要`TIME_WAIT`

主动方发送最后一次`ACK`之后进入`TIME_WAIT`状态，等待`2MSL`（两个报文最大生命周期），等待这段时间就是为了如果接收到了重发的`FIN`请求能够进行最后一次`ACK`回复，让在网络中延迟的`FIN/ACK`数据都消失在网络中，不会对后续连接造成影响。

##### b. 为什么是`2MSL`

保证客户端发送的最后一个`ACK`报文能够到达服务器，因为这个`ACK`报文可能丢失，站在服务器的角度看来，我已经发送了`FIN+ACK`报文请求断开了，客户端还没有给我回应，应该是我发送的请求断开报文它没有收到，于是服务器又会重新发送一次，而客户端就能在这个`2MSL`时间段内收到这个重传的报文，接着给出回应报文，并且会重启`2MSL`计时器。

##### c. 为什么是三次握手

3次握手完成两个重要的功能，既要双方做好发送数据的准备工作(双方都知道彼此已准备好)，也要允许双方就初始序列号进行协商，这个序列号在握手过程中被发送和确认。

##### d. 如果已经建立连接，但是客户端突然出现故障怎么办

TCP设有一个保活计时器，显然，客户端如果出现故障，服务器不能一直等下去，白白浪费资源。服务器每收到一次客户端的请求后都会重新复位这个计时器，时间通常是设置为2小时，若两小时还没有收到客户端的任何数据，服务器就会发送一个探测报文段，以后每隔75秒发送一次。若一连发送10个探测报文仍然没反应，服务器就认为客户端出了故障，接着就关闭连接。

#### socket 编程相关

##### a. 设置 Socket 缓冲区的大小

```c
int setsockopt(int sockfd, int level, int optname, const void *optval, socklen_t optlen);
```

通常，将`level`设置为`SOL_SOCKET`，将`optname`设置成`SO_RCVBUF`或者`SO_SNDBUF`。

#### TCP 可靠传输

1. 数据被分割成 TCP 认为最适合发送的数据块。
2. TCP 给发送的每一个包进行编号，接收方对数据包进行排序，把有序数据传送给应用层。
3. 校验和： TCP 将保持它首部和数据的检验和。这是一个端到端的检验和，目的是检测数据在传输过程中的任何变化。如果收到段的检验和有差错，TCP 将丢弃这个报文段和不确认收到此报文段。
4. TCP 的接收端会丢弃重复的数据。
5. 流量控制： TCP 连接的每一方都有固定大小的缓冲空间，TCP的接收端只允许发送端发送接收端缓冲区能接纳的数据。当接收方来不及处理发送方的数据，能提示发送方降低发送的速率，防止包丢失。TCP 使用的流量控制协议是可变大小的滑动窗口协议。 （TCP 利用滑动窗口实现流量控制）
6. 拥塞控制： 当网络拥塞时，减少数据的发送。
7. ARQ协议： 也是为了实现可靠传输的，它的基本原理就是每发完一个分组就停止发送，等待对方确认。在收到确认后再发下一个分组。
8. 超时重传： 当 TCP 发出一个段后，它启动一个定时器，等待目的端确认收到这个报文段。如果不能及时收到一个确认，将重发这个报文段。

##### a. ARQ 协议

###### 停止等待 ARQ 协议

- 停止等待协议是为了实现可靠传输的，它的基本原理就是每发完一个分组就停止发送，等待对方确认（回复ACK）。如果过了一段时间（超时时间后），还是没有收到 ACK 确认，说明没有发送成功，需要重新发送，直到收到确认后再发下一个分组；
- 在停止等待协议中，若接收方收到重复分组，就丢弃该分组，但同时还要发送确认；

###### 连续 ARQ 协议（回退 N 帧）

连续 ARQ 协议可提高信道利用率。发送方维持一个发送窗口，凡位于发送窗口内的分组可以连续发送出去，而不需要等待对方确认。接收方一般采用累计确认，对按序到达的最后一个分组发送确认，表明到这个分组为止的所有分组都已经正确收到了。


#### 3. 网络层

#### 网关协议

* IGP（内部网关协议）
  * RIP 协议，基于距离矢量算法，使用“跳数”衡量到达目标地址的路由距离。范围限制在15跳，使用UDP传输，520端口。
  * OSPF 协议，SPF算法用做生成最小生成树。是链路状态协议。
* EGP（外部网关协议）
  * BGP 协议，负责和其他的 BGP 系统交换网络可达信息。对等实体之间通过TCP（端口179）会话交互数据。

### 数据库

#### MySQL 高可用方案

##### a. 主从或主主半同步复制

![image-20200922092332462](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200922092332462.png)

##### b. 半同步复制优化

![image-20200922092719953](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200922092719953.png)

##### c. 高可用框架优化

![image-20200922092933947](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200922092933947.png)

##### d. 共享存储

<img src="C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200922093054884.png" alt="image-20200922093054884" style="zoom:50%;" />

##### e. 分布式协议

<img src="C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200922093159893.png" alt="image-20200922093159893" style="zoom:50%;" />

### 操作系统

#### 协程比线程执行效率高的原因

协程不是被操作系统内核所管理，而完全是由程序所控制（用户态内执行）。

* 协程的特点在于是一个线程执行。
* 协程的切换由程序自身控制，因此并没有线程切换的开销。

* 不需要多线程的锁机制，不存在同时写变量冲突，在协程中控制共享资源不加锁，只需要判断状态即可。

#### 虚拟内存

##### a. 产生的原因

* 物理内存有限，分配完之后，没有得到分配资源的进程就只能等待。
* 直接访问物理内存，可能会篡改其他进程的数据。

##### b. 地址访问过程

1. 每次我要访问地址空间上的某一个地址，都需要把地址翻译为实际物理内存地址。
2. 所有进程共享这整一块物理内存，每个进程只把自己目前需要的虚拟地址空间映射到物理内存上。
3. 进程需要知道哪些地址空间上的数据在物理内存上，这就需要通过页表来记录。
4. 页表的每一个表项分两部分，第一部分记录此页是否在物理内存上，第二部分记录物理内存页的地址（如果在的话）。
5. 当进程访问某个虚拟地址的时候，就会先去看页表，如果发现对应的数据不在物理内存上，就会发生缺页异常。
6. 缺页异常的处理过程，操作系统立即阻塞该进程，并将硬盘里对应的页换入内存，然后使该进程就绪，如果内存已经满了，没有空地方了，那就找一个页覆盖。

##### c. 虚拟内存的优点

1. 每个进程的内存空间都是一致而且固定的，链接器在链接可执行文件时，可以设定内存地址，而不用去管这些数据最终实际内存地址
2. 当不同的进程使用同一段代码时，比如库文件的代码，在物理内存中可以只存储一份这样的代码，不同进程只要将自己的虚拟内存映射过去即可，以节省物理内存
3. 在程序需要分配连续空间的时候，只需要在虚拟内存分配连续空间，而不需要物理内存时连续的，这样就可以有效地利用物理内存

##### d. 进程内存布局

- **文本段**：包含了进程运行的程序机器语言指令。文本段具有只读属性，以防止进程通过错误指针意外修改自身指令。因为多个进程可同时运行同一程序，所以又将文本段设为可共享，这样，一份程序代码的拷贝可以映射到所有这些进程的虚拟地址空间中。
- **初始化数据段**：包含显式初始化的全局变量和静态变量。当程序加载到内存时，从可执行文件中读取这些变量的值。
- **未初始化数据段**：包含了未进行显示初始化的全局变量和静态变量。程序启动之前，系统将本段内所有内存初始化为0。将经过初始化的全局变量和静态变量与未初始化的全局变量和静态变量分开存放，其主要原因在于程序在磁盘上存储时，没有必要为未经初始化的变量分配存储空间。
- **栈**（stack）：是一个动态增长和收缩的段，有栈帧（stack frames）组成。系统会为每个当前调用的函数分配一个栈帧。栈帧中存储了函数的局部变量（所谓自动变量）、实参和返回值。
- **堆**（heap）：是可在运行时（为变量）动态进行内存分配的一块区域。堆顶端称为program break。

##### e. 为什么用户态和内核态切换耗费时间

* 应用程序的执行必须依托于内核提供的资源，包括CPU资源、存储资源、I/O资源等。为了使上层应用能够访问到这些资源，内核必须为上层应用提供访问的接口：即系统调用。因此，如果一个程序需要从用户态进入内核态，那么它必须执行系统调用语句。

* 当程序中有系统调用语句，程序执行到系统调用时，首先使用软中断指令保存现场，然后去系统调用，在内核态执行后，再恢复现场，每个进程都会有两个栈，一个内核态栈和一个用户态栈。当中断执行时就会由用户态栈转向内核态栈。系统调用时需要进行栈的切换。而且内核代码对用户不信任，需要进行额外的检查。 

* 系统调用一般都需要保存用户程序的上下文(context), 在进入内核的时候需要保存用户态的寄存器，在内核态返回用户态的时候会恢复这些寄存器的内容。这是一个开销的地方。 如果需要在不同用户程序间切换的话，那么还要更新cr3寄存器，这样会更换每个程序的虚拟内存到物理内存映射表的地址，也是一个比较高负担的操作。

### Go 语言

#### 控制并发数量的方式

```go
func channel() {
    count, sum := 10, 100
    c := make(chan struct{}, count)
    sc := make(chan struct{}, sum)
    defer close(c)
    defer close(sc)
    for i:=0; i<sum; i++ { c <- struct{}{}  // 作用类似于 waitgroup.Add(1)
        go func(j int) { fmt.Println(j)  <- c
            sc <- struct{}{} }(i) }
    for i:=sum; i>0; i-- { <- sc } }
```

#### IO优化机制 netpoller

- 首先，无论何时，当在Go中打开或接收到一个链接时，其文件句柄都会被设为`NONBLOCKING`模式。
- 当调用相应的`Read/Write`等操作时，无论是否成功，都会直接返回而不会阻塞。当返回值是`EAGAIN`时，表示`IO`事件还没有到达，需要等待。这时，Go库函数调用`PollServer`的`AddFd()`将对应文件句柄加入`netpoller`的监控池，并将当前`Goroutine`阻塞。
- 当系统中存在空闲 `P & M` 时，`runtime`会首先查找本地就绪队列，若其空，则调用`netpoller`;` netpoller`通过OS提供的`epoll`或`kqueue`机制，检查已到达的IO事件，并唤醒对应的`Goroutine`返回给`runtime`，将其再度执行。
- 最后，`Goroutine`再次回到Go语言库上下文时，再调用`Read/Write`等IO操作时，就可以顺利返回了。

#### 如何阻塞一个 goroutine

```go
# 从一个不发送数据channel中接收数据
<-make(chan struct{}) 或者 <-make(<-chan struct{})
# 向不接收数据的channel中发送数据
make(chan struct{}) <- struct{}{} 或者 make(chan<- struct{}) <- struct{}{}
# 从空的channel中接收数据
<-chan struct{} (nil)
# 向空channel中发送数据
chan struct{}(nil) <- struct{}{}
# 使用select
select{}
```

#### 实现Timer定时器

##### a. time.After

```go
func main() {
	c := make(chan int)
	timeout := time.After(time.Second * 3)
	for { select { case d := <-c: fmt.Println(d)
		case <-timeout: fmt.Println("执行定时操作任务")
		case dd := <-time.After(time.Second * 3): fmt.Println(dd.Format("2006-01-02 15:04:05"), "执行超时动作") } } }
```

##### b. time.NewTimer

```go
func main() {
	t := time.NewTimer(time.Second * 2)
	ch := make(chan bool)
	go func(t *time.Timer) { defer t.Stop() for { select { case <-t.C:
				fmt.Println("timer running....")
				t.Reset(time.Second * 2)
			case stop := <-ch: if stop { fmt.Println("timer Stop")
					return } } } }(t)
	time.Sleep(10 * time.Second)
	ch <- true
	close(ch)
	time.Sleep(1 * time.Second) }
```

##### c. time.NewTicker

```go
func main() {
	ticker := time.NewTicker(2 * time.Second) 
	ch := make(chan bool)
	go func(ticker *time.Ticker) { defer ticker.Stop() for { select {
			case <-ticker.C: fmt.Println("Ticker running...")
			case stop := <-ch: if stop { fmt.Println("Ticker Stop")
					return } } } }(ticker)
	time.Sleep(10 * time.Second)
	ch <- true
	close(ch) }
```

### Linux

#### Linux 打开超大文件方法

```shel
# 查看文件的前 n 行
head -n /var/lib/mysql/slowquery.log > temp.log
# 查看文件的后 n 行
head -n /var/lib/mysql/slowquery.log > temp.log
# 查看文件的 n 到 m 行
sed -n 'n,mp' /var/lib/mysql/slowquery.log > temp.log
```

#### 如何查看文件大小

```shell
ls -lht 或者 ll
du -sh * 
du -s backup.sh 或者 ls -lh backup.sh 可以查看某一特定文件
```

#### 如何查看端口占用情况

```shell
netstat -anp | grep 3306
lsof -i:3306
```

