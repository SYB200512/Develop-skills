# gRPC初始

## 一、前提知识

gRPC 是 Google 开源的**高性能 RPC 框架**。

>
> RPC：Remote Procedure Call 远程过程调用。
> 简单理解：**像调用本地函数一样，调用另一台机器上的函数**。

传统 HTTP 接口（RESTful）：

- 基于 HTTP/1.1，传输格式 JSON（文本，体积大、序列化慢）
- 前后端用 URL 路径传参数，返回 JSON

gRPC：

- 默认使用 **HTTP/2** 底层传输
- 序列化用 **Protocol Buffers（protobuf）** 二进制编码，体积更小、速度更快
- 通过`.proto`文件**预先定义接口、请求结构体、返回结构体**，服务端客户端共用同一份接口定义

>
> 核心通信模型：你写的示例是 **一元调用（Unary RPC）**，最简单的一种。
> 一元 RPC：**客户端发 1 个请求 → 服务端返回 1 个响应**（类似普通 HTTP 接口，一次请求一次响应）

gRPC 一共有 4 种通信模型：

1. **Unary 一元 RPC（当前例子）**：1 请求 → 1 响应 
2. 服务端流式 RPC：客户端发 1 请求，服务端返回一连串流数据（如日志推送）
3. 客户端流式 RPC：客户端持续发多个请求，服务端最后返回 1 个响应（上传大文件）
4. 双向流式 RPC：两边互相持续收发消息（聊天实时通信）

## 二、Protocol Buffers(protobuf, .proto文件逐行解析)

```go
syntax = "proto3";  // 使用proto3版本，最常用，简洁
package user;       // 包名，用来隔离命名空间
option go_package = "./pb;pb"; 
// go_package = "生成文件输出目录;go里面的包名"
// ./pb 生成代码放到pb文件夹；第二个pb：生成go代码的package名字

// 定义gRPC服务：里面放一堆RPC方法
service UserService {
    // rpc 方法名(请求消息) returns (返回消息);
    rpc GetUser (GetUserRequest) returns (User);
    rpc ListUsers (ListUsersRequest) returns (ListUsersResponse);
    rpc CreateUser (CreateUserRequest) returns (User);
}

// message 等价Go里面的结构体！
message User {
    string id = 1;    // 1是字段编号！不是变量默认值，重点！
    string name = 2;
    string email = 3;
    int32 age = 4;
}

message GetUserRequest {
    string id = 1;
}

```

1. `message` = 结构体，用来定义**请求体 / 返回体**
2. string id = 1;
   - `1` 叫**字段标识号 (field number)**，protobuf 二进制序列化靠这个编号识别字段
   - ❗**不能随便修改编号！**一旦上线，编号固定。不能复用旧编号。
3. `service`：定义 RPC 服务，一组 rpc 接口。`rpc`就是远程方法

##### 生成 Go 代码命令

```go
protoc --go_out=. --go-grpc_out=. user.proto

```

- `--go_out=.`：生成 protobuf 序列化相关代码（message 结构体）
- `--go-grpc_out=.`：生成 gRPC 服务、客户端代码（service 对应的接口、client）

>
> 执行完命令，会在`./pb`目录生成`.pb.go`和`_grpc.pb.go`两个文件。
> 里面自动生成：
>
>
> - Go 结构体 `pb.User`、`pb.GetUserRequest`
> - 服务端需要实现的接口
> - 客户端调用的 Client 结构体

## 三、gRPC服务端代码解析

```go
type userServer struct {
    pb.UnimplementedUserServiceServer
}
//pb.UnimplementedUserServiceServer是protc自动生成的结构体，必须嵌入
//作用：如果新增rpc方法，没有实现，调用的时候直接返回未实现错误，不会编译报错

func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    // 业务逻辑
    return &pb.User{
        Id:    req.Id,
        Name:  "Alice",
        Email: "alice@example.com",
    }, nil
}
//参数：ctx context.Context：上下文，可以传递超时，元信息
//req *pb.GetUserRequest：客户端发来的请求参数，返回：*pb.User响应+error


func main() {
    lis, _ := net.Listen("tcp", ":50051")
    s := grpc.NewServer()
    pb.RegisterUserServiceServer(s, &userServer{})
    s.Serve(lis)
    //流程：监听端口-->创建grpc server -->注册自己写的服务实现-->启动服务等待客户端连接
    
}
```

## 四、gRPC客户端代码解析

```go
func main() {
    // 和grpc服务端建立长连接conn
    conn, _ := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
    defer conn.Close() // 程序结束关闭连接

    // 根据连接，生成UserService客户端对象
    client := pb.NewUserServiceClient(conn)

    // 直接调用远程方法GetUser，就像本地函数！
    user, _ := client.GetUser(context.Background(), &pb.GetUserRequest{Id: "1"})
    fmt.Printf("User: %+v\n", user)
}

```

1. `grpc.Dial`：建立 TCP 长连接（gRPC 基于 http2 长连接，复用连接，性能高）
2. `pb.NewUserServiceClient(conn)`：生成客户端，里面自动拥有所有 rpc 方法
3. `client.GetUser()`：调用远程函数，传入 context 和请求结构体，拿到返回 user ✅ 看起来像调用本地函数，但实际走网络到服务端执行！

> `insecure.NewCredentials()` 不开启 TLS 加密，开发测试用；生产环境必须用 TLS 证书加密。

## 五、完整调用流程

1. 客户端代码执行 `client.GetUser(..., req)`
2. 客户端 protobuf 把`GetUserRequest`序列化成二进制
3. 通过 HTTP/2 发送到 gRPC 服务端
4. 服务端收到二进制，protobuf 反序列化成`GetUserRequest`对象
5. 调用服务端 `GetUser`业务函数
6. 业务返回`pb.User`对象，protobuf 序列化为二进制
7. 传回客户端，客户端反序列化得到 User 结构体
8. 拿到结果，打印输出
