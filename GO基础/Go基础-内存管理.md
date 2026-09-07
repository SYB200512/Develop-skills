# Go基础-内存管理

## 1.1Go内存管理与分配

###### 1:栈vs堆

栈：栈是每个Goroutine私有的连续内存块，默认大小从Go1.4的4KB-1GB

特点：分配/释放开销低/无需GC参与/函数返回时自动释放/适合小对象，生命周期明确的变量



堆：堆是全局共享的内存区域，由Go的GC管理

特点：分配开销较大（需要锁，空闲列表查询）/需要GC追踪和回收/适合需要在函数间共享，生命周期不确定的             对象

```go
//栈上分配，变量生命周期与函数绑定
func stack() int{
    x := 31
    return x
}

//堆上分配，变量被外部引用
func heap() *int{
    x := 31
    return &x
}

```

###### 2:逃逸分析

Go编译器在编译时进行逃逸分析，决定变量分配在栈上还是堆上。逃逸分析的目标是：尽可能将变量分配在栈上

！可以用go  build  -gcflags="-m" 来查看逃逸分析结果

常见的逃逸场景：

1：返回指针--变量被函数返回引用--return &x

2：接口存储	将具体类型存入 interface{}	fmt.Println(x)

3：闭包捕获	闭包引用了外部变量	匿名函数引用外层变量

4：大对象	过大的栈上分配（>64KB）	make([]int, 100000)

5：全局变量	全局/包级变量	var g *int

```go
func example(){
    //接口存储
    var i interface{} = 8
    
    //闭包捕获
    t := 1
    nt := func(){
        t++
    }
    nt()
    
    //返回指针
    var l int
    p := &l
    *p = 22
    return p
    
}
```



###### 3.Go内存分配器（）

<img src="C:\Users\12472\AppData\Roaming\Typora\typora-user-images\image-20260902223213921.png" alt="image-20260902223213921" style="zoom:50%;" />

4.性能优化实践

对象复用

```
// 使用 sync.Pool 复用临时对象
var bufferPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 0, 4096)
    },
}

func processRequest() {
    buf := bufferPool.Get().([]byte)
    // 使用 buf...
    bufferPool.Put(buf[:0])  // 重置并归还
}
```

避免不必要的指针

```
//  使用指针（堆分配）
type Config struct {
    Name *string
    Port *int
}

// 使用值类型（栈分配）
type Config struct {
    Name string
    Port int
    Valid bool  // 用布尔标记零值有效性
}
```

###### 练习

1. 写一个函数，返回 `[]byte` 类型，用 `go build -gcflags="-m"` 检查逃逸情况。然后改为通过参数传入缓冲区的方式避免逃逸，对比两者的性能差异

```
// 版本1：函数内部自己 make []byte，返回，观察逃逸
func genData(n int) []byte {
	buf := make([]byte, 0, n)
	buf = append(buf, []byte("hello world, go escape test")...)
	return buf
}

// 版本2：传入外部缓冲区，复用传入的buf，避免函数内部分配
func genDataWithBuf(buf []byte) []byte {
	buf = buf[:0] // 重置长度，保留cap
	buf = append(buf, []byte("hello world, go escape test")...)
	return buf
}
```

