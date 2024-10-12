<h1>细说channel</h1>
<p>‍</p>
<h3>开门小菜</h3>
<pre><code>channel 是Go 语言中的一类特殊结构体，也是Go语言的特色之一。channel 管道 ，本质上是在堆上分配的缓冲区，用于传输数据，协程（Goroutine）之间通信使用。使用channel 能快速实现不同Goroutine之间的通信，正好适用高并发的环境
</code></pre>
<p>‍</p>
<p>‍</p>
<p>Channel 有缓冲区和无缓冲区的两种</p>
<p>对于有缓冲区的chanel来说：</p>
<ul>
<li>发送方会向缓冲区中写入数据，然后唤醒接收方，多个接收方会尝试从缓冲区中读取数据，如果没有读取到会重新陷入休眠；</li>
<li>接收方会从缓冲区中读取数据，然后唤醒发送方，发送方会尝试向缓冲区写入数据，如果缓冲区已满会重新陷入休眠；</li>
</ul>
<p>‍</p>
<h3>数据结构</h3>
<p>channel 对应<a href=""><code>runtime.hchan</code></a>​ 结构体</p>
<pre><code class="language-go">type hchan struct {
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
</code></pre>
<p>buf ：指向堆上分配的缓冲区的指针，（根据make 指定的大小分配）</p>
<p>‍</p>
<p>Channel 实际上是一个互斥的循环队列，（环形缓冲区）</p>
<p>‍</p>
<p>‍</p>
<h3>创建与关闭</h3>
<h4>创建</h4>
<pre><code class="language-go">   make(chan 元素类型, [缓冲大小])
</code></pre>
<p>若只是声明，则初始值为nil</p>
<p>‍</p>
<p>可以设置缓冲区大小，或者创建无缓冲区的Channel</p>
<p>‍</p>
<pre><code>这一阶段会对传入 `make`​ 关键字的缓冲区大小进行检查，如果我们不向 `make`​ 传递表示缓冲区大小的参数，那么就会设置一个默认值 0，也就是当前的 Channel 不存在缓冲区。
</code></pre>
<p>​<code>OMAKECHAN</code>​ 类型的节点最终都会在 SSA 中间代码生成阶段之前被转换成调用 <a href="https://draveness.me/golang/tree/runtime.makechan"><code>runtime.makechan</code></a>​ 或者 <a href="https://draveness.me/golang/tree/runtime.makechan64"><code>runtime.makechan64</code></a>​ 的函数：</p>
<h4>类型</h4>
<ul>
<li>
<p>普通的channel， 如 <code>channel chan int</code>​ 这种channel默认支持 发送和接收</p>
</li>
<li>
<p>只写channel ： <code>channel  chan &lt;- int </code>​</p>
</li>
<li>
<p>只读channel : <code>channel &lt;- chan  int </code>​</p>
</li>
</ul>
<p>普通的channel 类型就包含了下面的两种</p>
<p>‍</p>
<h4>关闭</h4>
<p>​<code>close(chan )</code>​</p>
<p>编译器会将用于关闭管道的 <code>close</code>​ 关键字转换成 <code>OCLOSE</code>​ 节点以及 <a href="https://draveness.me/golang/tree/runtime.closechan"><code>runtime.closechan</code></a>​ 函数。</p>
<p>当 Channel 是一个空指针或者已经被关闭时，Go 语言运行时都会直接崩溃并抛出异常：</p>
<p>‍</p>
<p>‍</p>
<p>‍</p>
<p>‍</p>
<p>‍</p>
<h3>用法</h3>
<h4>基本用法</h4>
<pre><code class="language-go">var channel chan int
</code></pre>
<ul>
<li>向channel中写入数据：</li>
</ul>
<pre><code class="language-go">channel &lt;- 666
</code></pre>
<ol>
<li>
<p>channel 缓冲区没满，或者存在协程在接收数据的时候不会阻塞</p>
</li>
<li>
<p>当channel 为nil ，接收队列为空（没有协程接受数据）或者没有缓冲区的时候发送数据会发生阻塞</p>
</li>
</ol>
<p>‍</p>
<ul>
<li>读取channel中的数据：</li>
</ul>
<p>将结果赋值给变量</p>
<pre><code class="language-go">a:= &lt;- channel
</code></pre>
<p>将结果丢弃</p>
<pre><code class="language-go">&lt;- channel
</code></pre>
<p>comma ok式风格</p>
<pre><code class="language-go">data,ok:= &lt;- channel
</code></pre>
<p>data默认为0，ok为false时channel已关闭</p>
<p>‍</p>
<pre><code>当channel 为nil ，发送队列为空（没有协程发送数据）或者缓冲区为空的时候接收数据会发生阻塞
</code></pre>
<p>‍</p>
<p>发送等待队列：当缓冲区已满会将还要发送数据的的Goroutine 放入发送等待队列</p>
<p>‍</p>
<p>编译转化：</p>
<pre><code>channel发送操作:  会被编译器转化为对`runtime.chansend1()`​的调用，然后调用`runtime.chansend()`​

select channel 的发送操作： 会被编译器转化为对`runtime.selectnbsend(chan,int)`​的调用，然后同样会调用`runtime.chansend()`​

channel 接收操作： 会被编译器转化为 `runtime.chanrecv1()`​或者是`runtime.chanrecv2()`​的调用，而继续调用`rutime.chanrecv()`​

select channel 接收操作:  会被编译器转化为 `runtime.selectnbrecv()`​或者是`runtime.selectnbrecv2()`​的调用，而继续调用`rutime.chanrecv()`​
</code></pre>
<p>‍</p>
<h4>进阶用法</h4>
<p>一般使用一个协程专门用Channel 来 一直接收数据或者是监听事件</p>
<p>有几种用法，看场景选择</p>
<ol>
<li>
<p>一直接收数据</p>
<pre><code class="language-go">dataChannel := make(chan int)
	// 启动一个协程来专门监听数据
	go func() {
		for data := range dataChannel {
			processData(data)
		}
		fmt.Println(&quot;Data channel closed, stopping listener.&quot;)
	}()
close(dataChannel)
</code></pre>
</li>
</ol>
<p>其中当 channel 关闭时候 for range 就会停止循环</p>
<ol start="2">
<li>监听事件</li>
</ol>
<p>使用for+select 的方式,如设置超时，关闭信号等等</p>
<p>超时控制：</p>
<pre><code class="language-go">func main() {
    ch := make(chan int)

    go func() {
        time.Sleep(2 * time.Second)
        ch &lt;- 1
    }()

    for {
        select {
        case msg := &lt;-ch:
            fmt.Println(&quot;Received:&quot;, msg)
            return
        case &lt;-time.After(1 * time.Second):
            fmt.Println(&quot;Timeout, no message received&quot;)
            return
        }
    }
}

</code></pre>
<ol start="3">
<li>
<p>限制启动协程数</p>
<pre><code class="language-go">
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
		limiter &lt;- struct{}{}
		go func(host string) {
			if PingAlive(host) {
				fmt.Printf(&quot;%v is alive\n&quot;, host)
				s.hosts = append(s.hosts, host)
			}
			Wg.Done()
			&lt;-limiter
		}(host)
	}
	Wg.Wait()
	close(limiter)
	return s.hosts, nil
}
</code></pre>
</li>
</ol>
<p>使用一个channel ，其中缓冲区长度就是启动最大的协程数</p>
<p>在启动协程前，先向channel中写入 空结构体 ，若是已经开启了足够数量的协程，那么就会阻塞等待其他协程完成任务才会启动协程</p>
<p>在启动协程前 <code>Wg.Add(1)</code>​ 加上一个锁</p>
<p>在完成任务后， channel中丢弃一个数据，使主协程继续执行，<code>Wg.Done()</code>​解锁</p>
<p>最后<code>Wg.Wait()</code>​ 等待所有协程完成任务，执行解锁操作</p>
<p>‍</p>
<h3>select</h3>
<p>类似于swtich case语句 。 用来选择 Channel 的接收或者发送</p>
<p>用法：</p>
<pre><code class="language-go">select {
case a:= &lt;- channel:
	...
default:
	...
}
</code></pre>
<pre><code>上面是非阻塞式的，当channel 没有阻塞的时候，选择一个case分支进行执行，当channel全部阻塞的时候，会执行default的代码

多路select 中 可以同时是接收channel中的数据，向channel中发送数据
</code></pre>
<p><span data-type="text" style="color: var(--b3-font-color3);">	其中default 可以不写</span>。select 可以搭配for循环使用</p>
<p>‍</p>
<p>‍</p>
<p>‍</p>
<p>原理：select 语句会被编译器转为对runtime.selectgo的函数调用</p>
<pre><code class="language-go">func selectgo(cas0 *scase, order0 *uint16,
 			pc0 *uintptr, nsends, nrecvs int, block bool) (int, bool)
</code></pre>
<p>case0：指向所有case分支的数组</p>
<p>order0： 一般是cas0大小的两倍，一部分用于轮询，一部分用于lock加锁顺序</p>
<p>‍</p>
<p>返回值，int: 执行了哪个分支  ；bool: channel是否关闭</p>
<p>‍</p>
<span data-type="text" style="color: var(--b3-font-color10);">select 实现的是：按序加锁、乱序轮询</span>、<span data-type="text" style="color: var(--b3-font-color10);">挂起等待、按序解锁、唤醒执行</span>、<span data-type="text" style="color: var(--b3-font-color10);">按序加锁、离开队列、按序解锁</span>
<p>‍</p>
<p>‍</p>
<p>‍</p>
<h3>应用</h3>
<p>使用Channel 传输数据</p>
<p>使用Channel 当作信号量</p>
<ul>
<li>扫描器，实现端口扫描（最多开启1w个协程）</li>
</ul>
<p>限制开启的协程数：</p>
<pre><code class="language-go">package scanner

import (
	&quot;fmt&quot;
	&quot;net&quot;
	&quot;time&quot;
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
	return &amp;Scanner{
		startPort:    1,
		endPort:      65535,
		target:       make([]string, 0),
		result:       make([]int, 0),
		maxGoroutine: 9000,
	}
}

func (s *Scanner) scanPort(host, scheme string, port int, res chan int) {
	addr := fmt.Sprintf(&quot;%v:%d&quot;, host, port)
	dial, err := net.DialTimeout(scheme, addr, 1*time.Second)
	if err != nil {
		res &lt;- 0
		return
	}
	err = dial.Close()
	res &lt;- port
}

func (s *Scanner) Scan(host string) []int {
	fmt.Println(&quot;start scan : &quot;, host)
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
		for i := s.startPort; i &lt;= s.endPort; i++ {
			semaphore &lt;- struct{}{} // 获取信号量 ,可以在这里阻塞
			go func(port int) {
				defer func() { &lt;-semaphore }() // 释放信号量,套一层，当channel 缓冲区满的时候，发生阻塞,ch
				s.scanPort(host, &quot;tcp&quot;, i, res)
			}(i)
		}
		for i := 0; i &lt; length; i++ {
			port := &lt;-res
			if port != 0 {
				fmt.Printf(&quot;port %d is open\n&quot;, port)
				s.result = append(s.result, port)
			}
		}
	}
	return s.result
}

func ScanPort(host string) []int {
	return DefaultScanner.Scan(host)
}
</code></pre>
<ul>
<li>fscan</li>
</ul>
<p>使用fscan 一个特点就是 先开启的 channel 监听，然后再启动goroutine</p>
<pre><code class="language-go">func PortScan(hostslist []string, ports string, timeout int64) []string {
	var AliveAddress []string
	probePorts := common.ParsePort(ports)
	if len(probePorts) == 0 {
		fmt.Printf(&quot;[-] parse port %s error, please check your port format\n&quot;, ports)
		return AliveAddress
	}
	noPorts := common.ParsePort(common.NoPorts)
	if len(noPorts) &gt; 0 {
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
	for i := 0; i &lt; workers; i++ {
		go func() {
			for addr := range Addrs {
				PortConnect(addr, results, timeout, &amp;wg)
				wg.Done()
			}
		}()
	}

	//添加扫描目标
	for _, port := range probePorts {
		for _, host := range hostslist {
			wg.Add(1)
			Addrs &lt;- Addr{host, port}
		}
	}
	wg.Wait()
	close(Addrs)
	close(results)
	return AliveAddress
}
</code></pre>
<p>‍</p>
