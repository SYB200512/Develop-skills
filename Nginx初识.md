# Nginx初识

## 一、回顾要点

1：静态网页：内容固定为服务器上的文件，无需后端处理，访问速度快，适合纯展示类的简单网页场景

2：动态网页：内容由后端程序结合数据库实时生成，具备较强的交互性和个性化能力，

3：正向代理：位于客户端与目标服务器之间，帮助客户端访问外部网络，同时隐藏真实客户端Ip，

4：反向代理：位于用户与后端服务器集群之间，接受请求并转发，隐藏真实服务器IP，同时提供安全防护和缓存加速功能

5：负载均衡：将用户请求智能分发到多台服务器，有效解决高并发访问压力，清除单点故障，实现系统的高可用性

## 二、nginx配置文件

###  配置层级结构

1.Main全局块：配置影响Nginx全局的指令

![image-20260901105330604](C:\Users\12472\AppData\Roaming\Typora\typora-user-images\image-20260901105330604.png)

2.Events&HTTPS核心块：Events管理网络连接，HTTP块包含所有web相关的核心配置，是Server块的父容器

![image-20260901105505553](C:\Users\12472\AppData\Roaming\Typora\typora-user-images\image-20260901105505553.png)

3.Server&Location业务块：Server定义虚拟主机，Location匹配具体路由路径，实现URL到资源的精准映射

![image-20260901105751240](C:\Users\12472\AppData\Roaming\Typora\typora-user-images\image-20260901105751240.png)

### 配置流程

![image-20260901105921516](C:\Users\12472\AppData\Roaming\Typora\typora-user-images\image-20260901105921516.png)

## 三、反向代理

反向代理是位于客户端与后端服务器之间的中介服务器，它负责接受客户端的所有请求，根据规则转发给后端真正处理业务的服务器，并将处理结果再返回给客户端

![image-20260901174221755](C:\Users\12472\AppData\Roaming\Typora\typora-user-images\image-20260901174221755.png)

## 四、负载均衡

负载均衡作为流量分发的核心枢纽，是现代高可用架构的基石。通过智能调度策略管理服务器集群，解决了单节点性能瓶颈，构建了系统的高可用，高性能与可拓展三大核心。

![image-20260901195350315](C:\Users\12472\AppData\Roaming\Typora\typora-user-images\image-20260901195350315.png)