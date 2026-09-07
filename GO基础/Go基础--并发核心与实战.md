# Go基础--并发核心与实战

## 1.2：并发编程核心

### 1.Goroutine与GMP模型

##### G (Goroutine)：协程，任务本体

- Go 的轻量级执行单元，就是我们写 `go func(){}` 创建的东西。
- 保存协程上下文：栈、程序计数器 PC、状态、任务函数。
- 初始栈很小（2KB），栈可以动态扩容 / 缩容，成千上万个 G 开销很小。
- G 的几种状态：
  - `Gidle`：空闲，还没运行
  - `Grunnable`：就绪，放在运行队列，等待被调度
  - `Grunning`：正在 M 上运行
  - `Gsyscall`：陷入系统调用，阻塞
  - `Gwaiting`：阻塞（channel、锁、sleep）
  - `Gdead`：执行结束

>
> G 本身不具备执行能力，必须绑定 M 才能跑。

##### M (Machine)：操作系统线程

- 对应内核 OS 线程，由操作系统调度。
- M 是真正执行代码的载体；M 必须拿到 P，才能执行 G。
- M 的数量：默认和内核线程数不绑定，受 `GOMAXPROCS` 间接影响；发生 syscall 阻塞时会新建 M。
- M 保存 OS 线程栈、寄存器。

>
> M 多了开销大，内核线程成本高，Go 不会无脑开大量 M。

##### P (Processor)：调度器逻辑处理器，调度中介

- P 是 GMP 的核心，**调度上下文**，不是线程。
- P 的数量由环境变量 `GOMAXPROCS` 决定，默认等于 CPU 核心数。
- P 持有：本地运行队列（runq）、内存分配器缓存、调度相关资源。
- 规则：**M 想要执行 G，必须绑定一个 P；一个 P 同一时间只能绑定一个 M**。

>
> P 控制并行度：最多同时有 GOMAXPROCS 个 G 在 CPU 上并行执行。

##### 两种队列

1. **P 本地队列 runq**：每个 P 自己的队列，存放待运行 G，优先从本地取 G，锁竞争小。
2. **全局队列 GQ**：所有 P 共享，加全局锁；本地队列满了 G 会丢到全局队列，P 本地没有任务时去全局队列偷 G。

**总结**：P 负责调度，M 负责执行，G 是待执行任务；M 必须持有 P 才能跑 G；阻塞 syscall 时 M 和 P 分离，通过工作窃取均衡任务，实现 M:N 并发调度

### 2.Channel使用模式

 channel是Go协程之间通信的核心，遵循不要通过共享内存通信，要通过通信共享内存

基操：

```go
ch := make(chan int)        //无缓冲channel(同步)
nch := make(chan int , 5)  //有缓冲channel(异步)

ch <- 32 //发送
val,ok := <- ch //接收
close(ch) // 关闭
```

###### 核心模式-1：生产者-消费者

核心：一组生产者往 channel 生产数据；一组消费者从 channel 取数据处理。生产者关闭 channel，消费者 range 退出

```go
// 生产者
func producer(ch chan<- int, wg *sync.WaitGroup) {
	defer wg.Done()
	for i := 0; i < 5; i++ {
		ch <- i
	}
}
// 消费者
func consumer(ch <-chan int, wg *sync.WaitGroup) {
	defer wg.Done()
	for v := range ch {
		fmt.Printf("消费：%d\n", v)
	}
}

func main() {
	ch := make(chan int, 2)
	var wgPro sync.WaitGroup
	var wgCon sync.WaitGroup

	// 2个生产者
	for i := 0; i < 2; i++ {
		wgPro.Add(1)
		go producer(ch, &wgPro)
	}

	// 协程等待所有生产者完成后关闭channel
	go func() {
		wgPro.Wait()
		close(ch)
	}()

	// 3个消费者
	for i := 0; i < 3; i++ {
		wgCon.Add(1)
		go consumer(ch, &wgCon)
	}
	wgCon.Wait()
	fmt.Println("全部完成")
}

```

###### 核心模式-1：工作池（协程池）

限制并发数量：任务丢进任务 channel，固定 N 个 worker goroutine，从 channel 获取任务执行，控制最大并发数。

```go
type Task struct {
	ID int
}

func worker(id int, taskCh <-chan Task, wg *sync.WaitGroup) {
	defer wg.Done()
	for t := range taskCh {
		fmt.Printf("worker %d 处理任务 %d\n", id, t.ID)
	}
}

func main() {
	const workerNum = 3 // 最多3个并发
	taskCh := make(chan Task, 10)
	var wg sync.WaitGroup

	// 启动固定数量 worker
	for i := 0; i < workerNum; i++ {
		wg.Add(1)
		go worker(i, taskCh, &wg)
	}

	// 投放任务
	for i := 0; i < 10; i++ {
		taskCh <- Task{ID: i}
	}
	close(taskCh) // 任务全部提交完毕关闭channel，worker range退出

	wg.Wait()
	fmt.Println("所有任务处理完毕")
```

关键点：

1. workerNum 控制并发上限，防止无限创建 goroutine 打满资源
2. 任务全部入队后 close (taskCh)，worker 通过 range 结束
3. wg 等待全部 worker 退出

###### 核心模式-3：管道Pipeline

流水线：多个阶段，每个阶段 goroutine，channel 作为阶段之间数据流，数据依次流经 stage1 → stage2 → stage3。每个阶段：接收上游 channel，处理，输出到下游 channel。

```
//阶段 1：生成数字
//阶段 2：数字平方
//阶段 3：打印结果
// stage1：生成数据
func gen(nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		for _, n := range nums {
			out <- n
		}
		close(out)
	}()
	return out
}

// stage2：平方处理
func square(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		for v := range in {
			out <- v * v
		}
		close(out)
	}()
	return out
}

// stage3：消费输出
func print(in <-chan int) {
	for v := range in {
		fmt.Println(v)
	}
}

func main() {
	ch := gen(1,2,3,4)
	ch = square(ch)
	print(ch)
}
```

### 3.Select多路复用

`select` 同时等待多个 Channel 操作：

```
func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() {
        time.Sleep(1 * time.Second)
        ch1 <- "one"
    }()
    go func() {
        time.Sleep(2 * time.Second)
        ch2 <- "two"
    }()

    select {
    case msg := <-ch1:
        fmt.Println("Received from ch1:", msg)
    case msg := <-ch2:
        fmt.Println("Received from ch2:", msg)
    case <-time.After(3 * time.Second):
        fmt.Println("Timeout")
    }
}
```

### 4.Context

`context.Context` 是 Go 并发中的核心接口，用于传递取消信号和请求范围的值：

```
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

```go
// 根 Context
ctx := context.Background()

// 可取消
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

// 超时
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// 截止时间
ctx, cancel := context.WithDeadline(context.Background(),time.Now().Add(5*time.Second))
defer cancel()

// 传值
ctx := context.WithValue(context.Background(), "key", "value")
```

### 5.并发安全原则

不要通过共享内存来通信，而要通过通信来共享内存

```
// ❌ 错误：共享内存 + 互斥锁
type Counter struct {
    mu    sync.Mutex
    value int
}

func (c *Counter) Incr() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}

// ✅ 正确：通过 Channel 通信
type Counter struct {
    value int
}

func (c *Counter) Run() <-chan int {
    out := make(chan int)
    go func() {
        for i := 0; i < 100; i++ {
            c.value++
            out <- c.value
        }
        close(out)
    }()
    return out
}
```

### 练习

1. 用 select + context.WithTimeout 实现一个「超时优先」模式：两个 Goroutine 并发查询同一数据，使用最快的那个结果，另一个超时取消。

```go
package main

import (
	"context"
	"fmt"
)

// 真实查询函数，业务逻辑写这里，ctx用于接收取消信号
func query(ctx context.Context, name string) (string, error) {
	// 这里替换成真实逻辑：http请求 / rpc / db查询
	// 一定要把ctx透传给下游调用！
	// eg: http.NewRequestWithContext(ctx,"GET",url,nil)

	// 如果内部有循环/阻塞逻辑，需要监听 ctx.Done()
	select {
	case <-ctx.Done():
		return "", ctx.Err()
	default:
	}

	if name == "A" {
		return "result_A", nil
	}
	return "result_B", nil
}

func fastQuery(timeout time.Duration) (string, error) {
	ctx, cancel := context.WithTimeout(context.Background(), timeout)
	defer cancel()

	resCh := make(chan string, 1)
	errCh := make(chan error, 2)

	// 协程A
	go func() {
		res, err := query(ctx, "A")
		if err != nil {
			errCh <- err
			return
		}
		resCh <- res
	}()

	// 协程B
	go func() {
		res, err := query(ctx, "B")
		if err != nil {
			errCh <- err
			return
		}
		resCh <- res
	}()

	select {
	case r := <-resCh:
		return r, nil
	case <-ctx.Done():
		return "", fmt.Errorf("timeout: %w", ctx.Err())
	}
}

func main() {
	res, err := fastQuery(2 * time.Second)
	if err != nil {
		fmt.Println("err:", err)
	} else {
		fmt.Println("fast result:", res)
	}
}
```

1. 实现一个简单的扇出（Fan-Out）模式：一个生产者向 3 个 worker 分发任务，每个 worker 处理完毕后将结果发送到同一个结果 Channel。

```go
package main

import (
	"fmt"
	"sync"
)
// 任务
type Task struct {
	ID int
}
// 处理后的结果
type Result struct {
	TaskID int
	Data   string
}
// worker：从taskCh取任务，处理，发送结果到resultCh
func worker(id int, taskCh <-chan Task, resultCh chan<- Result, wg *sync.WaitGroup) {
	defer wg.Done()
	for t := range taskCh {
		// 执行业务处理
		res := Result{
			TaskID: t.ID,
			Data:   fmt.Sprintf("worker%d处理task%d", id, t.ID),
		}
		resultCh <- res
	}
}

func main() {
	const workerCount = 3
	taskCh := make(chan Task, 5)
	resultCh := make(chan Result, 5)

	var wg sync.WaitGroup

	// 启动3个worker（扇出）
	for i := 0; i < workerCount; i++ {
		wg.Add(1)
		go worker(i, taskCh, resultCh, &wg)
	}
	// 生产者：写入任务
	tasks := []Task{{ID: 1}, {ID: 2}, {ID: 3}, {ID: 4}, {ID: 5}}
	for _, t := range tasks {
		taskCh <- t
	}
	close(taskCh) // 任务全部投递完毕，关闭任务通道，worker的range可以退出
	// 另起协程等待所有worker完成，然后关闭结果通道
	go func() {
		wg.Wait()
		close(resultCh)
	}()
	// 消费结果通道，扇入：收集所有worker输出
	for r := range resultCh {
		fmt.Printf("收到结果：%+v\n", r)
	}

	fmt.Println("全部任务处理完成")
}
```

