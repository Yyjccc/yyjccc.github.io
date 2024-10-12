---
title: 细说channel
date: '2024-09-03 21:51:13'
updated: '2024-10-12 14:52:39'
permalink: /post/speaking-carefully-1ru2ii.html
comments: true
toc: true
---

# 细说channel

‍

### 开门小菜

	channel 是Go 语言中的一类特殊结构体，也是Go语言的特色之一。channel 管道 ，本质上是在堆上分配的缓冲区，用于传输数据，协程（Goroutine）之间通信使用。使用channel 能快速实现不同Goroutine之间的通信，正好适用高并发的环境

‍

‍

Channel 有缓冲区和无缓冲区的两种

对于有缓冲区的chanel来说：

* 发送方会向缓冲区中写入数据，然后唤醒接收方，多个接收方会尝试从缓冲区中读取数据，如果没有读取到会重新陷入休眠；
* 接收方会从缓冲区中读取数据，然后唤醒发送方，发送方会尝试向缓冲区写入数据，如果缓冲区已满会重新陷入休眠；

‍

### 数据结构

channel 对应[`runtime.hchan`]()​ 结构体

```go
type hchan struct {
	qcount   uint	//Channel 中的元素个数；
	dataqsiz uint	//Channel 中的循环队列的长度；
	buf      unsafe.Pointer // Channel 的缓冲区数据指针；
	elemsize uint16			//当前 Channel 能够收发的元素的大小
	closed   uint32		//是否关闭的状态
	elemtype *_type		//当前 Channel 能够收发的元素类型
	sendx    uint		// Channel 的发送操作处理到的位置；
	recvx    uint		//  Channel 的接收操作处理到的位置；
	recvq    waitq		//  接收队列，包含等待接收 channel 数据的 goroutines 的信息
	sendq    waitq		//  发送队列，包含等待发送 channel 数据的 goroutines 的信息 

	lock mutex			//互斥锁
}
```

buf ：指向堆上分配的缓冲区的指针，（根据make 指定的大小分配）

‍

Channel 实际上是一个互斥的循环队列，（环形缓冲区）

‍

‍

### 创建与关闭

#### 创建

```go
   make(chan 元素类型, [缓冲大小])
```

若只是声明，则初始值为nil

‍

可以设置缓冲区大小，或者创建无缓冲区的Channel

‍

	这一阶段会对传入 `make`​ 关键字的缓冲区大小进行检查，如果我们不向 `make`​ 传递表示缓冲区大小的参数，那么就会设置一个默认值 0，也就是当前的 Channel 不存在缓冲区。

​`OMAKECHAN`​ 类型的节点最终都会在 SSA 中间代码生成阶段之前被转换成调用 [`runtime.makechan`](https://draveness.me/golang/tree/runtime.makechan)​ 或者 [`runtime.makechan64`](https://draveness.me/golang/tree/runtime.makechan64)​ 的函数：

#### 类型

* 普通的channel， 如 `channel chan int`​ 这种channel默认支持 发送和接收

* 只写channel ： `channel  chan <- int `​
* 只读channel : `channel <- chan  int `​

普通的channel 类型就包含了下面的两种

‍

#### 关闭

​`close(chan )`​

编译器会将用于关闭管道的 `close`​ 关键字转换成 `OCLOSE`​ 节点以及 [`runtime.closechan`](https://draveness.me/golang/tree/runtime.closechan)​ 函数。

当 Channel 是一个空指针或者已经被关闭时，Go 语言运行时都会直接崩溃并抛出异常：

‍

‍

‍

‍

‍

### 用法

#### 基本用法

```go
var channel chan int
```

* 向channel中写入数据：

```go
channel <- 666
```

1. channel 缓冲区没满，或者存在协程在接收数据的时候不会阻塞

2. 当channel 为nil ，接收队列为空（没有协程接受数据）或者没有缓冲区的时候发送数据会发生阻塞

‍

* 读取channel中的数据：

将结果赋值给变量

```go
a:= <- channel
```

将结果丢弃

```go
<- channel
```

comma ok式风格

```go
data,ok:= <- channel
```

data默认为0，ok为false时channel已关闭

‍

	当channel 为nil ，发送队列为空（没有协程发送数据）或者缓冲区为空的时候接收数据会发生阻塞

‍

发送等待队列：当缓冲区已满会将还要发送数据的的Goroutine 放入发送等待队列

‍

编译转化：

	channel发送操作:  会被编译器转化为对`runtime.chansend1()`​的调用，然后调用`runtime.chansend()`​

 	select channel 的发送操作： 会被编译器转化为对`runtime.selectnbsend(chan,int)`​的调用，然后同样会调用`runtime.chansend()`​

	channel 接收操作： 会被编译器转化为 `runtime.chanrecv1()`​或者是`runtime.chanrecv2()`​的调用，而继续调用`rutime.chanrecv()`​

	select channel 接收操作:  会被编译器转化为 `runtime.selectnbrecv()`​或者是`runtime.selectnbrecv2()`​的调用，而继续调用`rutime.chanrecv()`​

‍

#### 进阶用法

一般使用一个协程专门用Channel 来 一直接收数据或者是监听事件

有几种用法，看场景选择

1. 一直接收数据

    ```go
    dataChannel := make(chan int)
    	// 启动一个协程来专门监听数据
    	go func() {
    		for data := range dataChannel {
    			processData(data)
    		}
    		fmt.Println("Data channel closed, stopping listener.")
    	}()
    close(dataChannel)
    ```

其中当 channel 关闭时候 for range 就会停止循环

2. 监听事件

使用for+select 的方式,如设置超时，关闭信号等等

超时控制：

```go
func main() {
    ch := make(chan int)

    go func() {
        time.Sleep(2 * time.Second)
        ch <- 1
    }()

    for {
        select {
        case msg := <-ch:
            fmt.Println("Received:", msg)
            return
        case <-time.After(1 * time.Second):
            fmt.Println("Timeout, no message received")
            return
        }
    }
}

```

3. 限制启动协程数

    ```go

    // 扫描c段主机
    func (s *Scanner) ScanHostsC(host string) ([]string, error) {
    	ips, err := getCClassIPs(host)
    	if err != nil {
    		fmt.Println(err)
    		return nil, err
    	}
    	var Wg sync.WaitGroup
    	//限制50个协程
    	limiter := make(chan struct{}, 50)
    	for _, host := range ips {
    		Wg.Add(1)
    		limiter <- struct{}{}
    		go func(host string) {
    			if PingAlive(host) {
    				fmt.Printf("%v is alive\n", host)
    				s.hosts = append(s.hosts, host)
    			}
    			Wg.Done()
    			<-limiter
    		}(host)
    	}
    	Wg.Wait()
    	close(limiter)
    	return s.hosts, nil
    }
    ```

使用一个channel ，其中缓冲区长度就是启动最大的协程数

在启动协程前，先向channel中写入 空结构体 ，若是已经开启了足够数量的协程，那么就会阻塞等待其他协程完成任务才会启动协程

在启动协程前 `Wg.Add(1)`​ 加上一个锁

在完成任务后， channel中丢弃一个数据，使主协程继续执行，`Wg.Done()`​解锁

最后`Wg.Wait()`​ 等待所有协程完成任务，执行解锁操作

‍

### select

类似于swtich case语句 。 用来选择 Channel 的接收或者发送

用法：

```go
select {
case a:= <- channel:
	...
default:
	...
}
```

	上面是非阻塞式的，当channel 没有阻塞的时候，选择一个case分支进行执行，当channel全部阻塞的时候，会执行default的代码

	多路select 中 可以同时是接收channel中的数据，向channel中发送数据

<span data-type="text" style="color: var(--b3-font-color3);">	其中default 可以不写</span>。select 可以搭配for循环使用

‍

‍

‍

原理：select 语句会被编译器转为对runtime.selectgo的函数调用

```go
func selectgo(cas0 *scase, order0 *uint16,
 			pc0 *uintptr, nsends, nrecvs int, block bool) (int, bool)
```

case0：指向所有case分支的数组

order0： 一般是cas0大小的两倍，一部分用于轮询，一部分用于lock加锁顺序

‍

返回值，int: 执行了哪个分支  ；bool: channel是否关闭

‍

<span data-type="text" style="color: var(--b3-font-color10);">select 实现的是：按序加锁、乱序轮询</span>、<span data-type="text" style="color: var(--b3-font-color10);">挂起等待、按序解锁、唤醒执行</span>、<span data-type="text" style="color: var(--b3-font-color10);">按序加锁、离开队列、按序解锁</span>

‍

‍

‍

### 应用

使用Channel 传输数据

使用Channel 当作信号量

* 扫描器，实现端口扫描（最多开启1w个协程）

限制开启的协程数：

```go
package scanner

import (
	"fmt"
	"net"
	"time"
)

var DefaultScanner = NewScanner()

type Scanner struct {
	startPort    int
	endPort      int
	target       []string
	host         string
	result       []int
	maxGoroutine int
}

func NewScanner() *Scanner {
	return &Scanner{
		startPort:    1,
		endPort:      65535,
		target:       make([]string, 0),
		result:       make([]int, 0),
		maxGoroutine: 9000,
	}
}

func (s *Scanner) scanPort(host, scheme string, port int, res chan int) {
	addr := fmt.Sprintf("%v:%d", host, port)
	dial, err := net.DialTimeout(scheme, addr, 1*time.Second)
	if err != nil {
		res <- 0
		return
	}
	err = dial.Close()
	res <- port
}

func (s *Scanner) Scan(host string) []int {
	fmt.Println("start scan : ", host)
	var length int
	flag := len(s.target) == 0
	if flag {
		length = s.endPort - s.startPort + 1
	} else {
		length = len(s.target)
	}
	res := make(chan int, length)
	defer close(res)
	// 创建一个带缓冲的通道，用于控制并发协程的数量
	semaphore := make(chan struct{}, s.maxGoroutine)
	if flag {
		for i := s.startPort; i <= s.endPort; i++ {
			semaphore <- struct{}{} // 获取信号量 ,可以在这里阻塞
			go func(port int) {
				defer func() { <-semaphore }() // 释放信号量,套一层，当channel 缓冲区满的时候，发生阻塞,ch
				s.scanPort(host, "tcp", i, res)
			}(i)
		}
		for i := 0; i < length; i++ {
			port := <-res
			if port != 0 {
				fmt.Printf("port %d is open\n", port)
				s.result = append(s.result, port)
			}
		}
	}
	return s.result
}

func ScanPort(host string) []int {
	return DefaultScanner.Scan(host)
}
```

* fscan

使用fscan 一个特点就是 先开启的 channel 监听，然后再启动goroutine

```go
func PortScan(hostslist []string, ports string, timeout int64) []string {
	var AliveAddress []string
	probePorts := common.ParsePort(ports)
	if len(probePorts) == 0 {
		fmt.Printf("[-] parse port %s error, please check your port format\n", ports)
		return AliveAddress
	}
	noPorts := common.ParsePort(common.NoPorts)
	if len(noPorts) > 0 {
		temp := map[int]struct{}{}
		for _, port := range probePorts {
			temp[port] = struct{}{}
		}

		for _, port := range noPorts {
			delete(temp, port)
		}

		var newDatas []int
		for port := range temp {
			newDatas = append(newDatas, port)
		}
		probePorts = newDatas
		sort.Ints(probePorts)
	}
	workers := common.Threads
	Addrs := make(chan Addr, 100)
	results := make(chan string, 100)
	var wg sync.WaitGroup

	//接收结果
	go func() {
		for found := range results {
			AliveAddress = append(AliveAddress, found)
			wg.Done()
		}
	}()

	//多线程扫描
	for i := 0; i < workers; i++ {
		go func() {
			for addr := range Addrs {
				PortConnect(addr, results, timeout, &wg)
				wg.Done()
			}
		}()
	}

	//添加扫描目标
	for _, port := range probePorts {
		for _, host := range hostslist {
			wg.Add(1)
			Addrs <- Addr{host, port}
		}
	}
	wg.Wait()
	close(Addrs)
	close(results)
	return AliveAddress
}
```

‍
