# Go1.23+新特性

#### 迭代器（就是遍历访问一下数据，不做处理）

```go
// 自定义迭代器
func Backward[E any](s []E) func(func(int, E) bool) {
    return func(yield func(int, E) bool) {
        for i := len(s) - 1; i >= 0; i-- {
            if !yield(i, s[i]) {
                return
            }
        }
    }
}

func main() {
    s := []string{"a", "b", "c"}
    for i, v := range Backward(s) {
        fmt.Println(i, v) // 2 c, 1 b, 0 a
    }
}
```

## time包改进

```go
//新的时间比较方法
t1.After(t2) //bool
t1.Before(t2) //bool
t1.Compare(t2) //-1,0,1
```

## cryto/rand增强-随机生成数

```go
//统一的随机数API
import "cryto/rand"

buf := make([]byte,16)
_,err := rand.Read(buf)
```

## 泛型类型别名

```go
//Go 1.24支持泛型类型别名
type Vector[T any] = []T

type Func[T any, R any] = func(T) R

//泛型别名与原始类型完全兼容
var v Vector[int] = []int{1,2,3}
```

