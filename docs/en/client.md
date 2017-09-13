# Example

[client-side code](http://icode.baidu.com/repo/baidu/opensource/baidu-rpc/files/master/blob/example/echo_c++/client.cpp) of echo.

# Quick facts

- Channel.Init() is not thread-safe.
- Channel.CallMethod() is thread-safe and a Channel can be unused by multiple threads simultaneously.
- Channel can be allocated on stack.
- Channel can be destructed after sending asynchronous request.
- No class named brpc::Client.

# Channel
Client-side sends requests. It's called [Channel](http://icode.baidu.com/repo/baidu/opensource/baidu-rpc/files/master/blob/src/brpc/channel.h) rather "Client" in brpc. A channel may represent a communication line to one server or multiple servers, which can be used to call services. 

A Channel can be **shared by all threads** in the process. Yon don't need to create separate Channels for each thread, and you don't need to synchronize Channel.CallMethod with lock. However creation and destroying of Channel is **not** thread-safe,  make sure the channel is initialized and destroyed only by one thread.

Some RPC implementations have so-called "ClientManager", including configurations and resource management at the client-side, which is not needed by brpc. "thread-num", "connection-type" such parameters are either in brpc::ChannelOptions or global gflags. Advantages of doing so:

1. Convenience. You don't have to pass a "ClientManager" when the Channel is created, and you don't have to store the "ClientManager". Otherwise many code have to pass "ClientManager" layer by layer, which is troublesome. gflags makes configurations of global behaviors easier.
2. Share resources. For example, servers and channels in brpc share background threads (workers of bthread).
3. Better management of Lifetime. Destructing a "ClientManager" is very error-prone, which is managed by brpc right now.

Like most classes, Channel must be **Init()**-ed before usage. Parameters take default values when `options` is NULL. If you want to use non-default values, code as follows:
```c++
brpc::ChannelOptions options;  // including default values
options.xxx = yyy;
...
channel.Init(..., &options);
```
Note that Channel neither modifies `options` nor accesses `options` after completion of Init(), thus options can be put on stack safely as in above code. Channel.options() gets options being used by the Channel.

Init() can connect one server or multiple servers(a cluster).

# Connect a server

```c++
// Take default values when options is NULL.
int Init(EndPoint server_addr_and_port, const ChannelOptions* options);
int Init(const char* server_addr_and_port, const ChannelOptions* options);
int Init(const char* server_addr, int port, const ChannelOptions* options);
```
The server connected by these Init() has fixed IP address genrally. The creation does not need NamingService and LoadBalancer, being relatively light-weight.  Don't frequently create Channels connecting to a hostname, which requires a DNS lookup and may take at most 10 seconds. (the default timeout of DNS lookup).

Valid "server_addr_and_port":
- 127.0.0.1:80
- cq01-cos-dev00.cq01.baidu.com:8765
- localhost:9000

Invalid "server_addr_and_port": 
- 127.0.0.1:90000     # too large port
- 10.39.2.300:8000   # invalid ip

# Connect a cluster

```c++
int Init(const char* naming_service_url,
         const char* load_balancer_name,
         const ChannelOptions* options);
```
Channels created by those Init() get server list from the NamingService specified by `naming_service_url` periodically or driven-by-events, and send request to one server chosen from the list according to the algorithm specified by `load_balancer_name` . 

You **should not** create such channels ad-hocly each time before a RPC, because creation and destroying of such channels relate to more resources, say NamingService needs to be accessed once at creation otherwise servers to connection are unknown. On the other hand, channels can be shared by multiple threads safely and has no need to be created frequently.

If `load_balancer_name` is NULL or empty, this Init() is just the one for single server and `naming_service_url` should be "ip:port" or "host:port" of the server. You can unify initialization of all channels with this Init(). For example, you can put values of `naming_service_url` and `load_balancer_name` in the configuration file, set `load_balancer_name` to empty for a single server and a valid algorithm for a cluster.

## Naming Service

Naming service maps a name to a modifiable list of servers, it's positioned as follows at client-side: 

![img](../images/ns.png)

With the help of naming service, the client remembers a name instead of every concreate servers. When the servers are added or removed, only the mapping in the naming service is changed, rather than modifying every client that may access the cluster. This process is called "decoupling upstreams and downstreams". General form of `naming_service_url`  is "**protocol://service_name**".

### bns://\<bns-name\>

BNS is the common naming service inside Baidu. For example: in "bns://rdev.matrix.all", "bns" is protocol and "rdev.matrix.all" is service-name. A related gflag is -ns_access_interval: ![img](../images/ns_access_interval.png)

If BNS displays non-empty list, but Channel says "no servers", the status bit of the machine in BNS is probably non-zero, meaning the machine is unavailable and as a correspondence not added as server candidates of the Channel. Status bits can be viewed by: 

`get_instance_by_service [bns_node_name] -s`

### file://\<path\>

Servers are put in the file specified by `path`, for example: in "file://conf/local_machine_list", "conf/local_machine_list" is the file and each line of the file should be address of a server. brpc reloads the file when it's updated.

### list://\<addr1\>,\<addr2\>...

Servers are directly put after list://, separated by comma. For example: "list://db-bce-81-3-186.db01:7000,m1-bce-44-67-72.m1:7000,cp01-rd-cos-006.cp01:7000" has 3 addresses.

### http://\<url\>

Connect all servers under the domain, for example: http://www.baidu.com:80. Note that although the Init() for a single server(2 arguments) accepts hostname as well, it only connects one server under the domain.

### Naming Service Filter

Users can filter servers got from the NamingService before sending to LoadBalancer.

![img](../images/ns_filter.jpg)

Interface of the filter: 
```c++
// naming_service_filter.h
class NamingServiceFilter {
public:
    // Return true to take this `server' as a candidate to issue RPC
    // Return false to filter it out
    virtual bool Accept(const ServerNode& server) const = 0;
};
 
// naming_service.h
struct ServerNode {
    butil::EndPoint addr;
    std::string tag;
};
```
The most common usage is filtering by server tags.

Customized filter is set to ChannelOptions and being NULL by default (not filter)

```c++
class MyNamingServiceFilter : public brpc::NamingServiceFilter {
public:
    bool Accept(const brpc::ServerNode& server) const {
        return server.tag == "main";
    }
};
 
int main() {
    ...
    MyNamingServiceFilter my_filter;
    ...
    brpc::ChannelOptions options;
    options.ns_filter = &my_filter;
    ...
}
```

## Load Balancer

When there're more than one servers to access, we need to divide the traffic. The process is often called load balancing, which is positioned as follows at client-side.

![img](../images/lb.png)

The ideal algorithm is to make every request being processed in-time, and crash of any server makes minimal impact. However clients cannot know delays or congestions happened at servers in realtime, and load balancing algorithms should be light-weight generally, users need to choose proper algorithms for their use cases. Algorithms provided by brpc right now (specified by `load_balancer_name`): 

### rr

which is round robin. Always choose the next server inside the list, the next of the last server is the first one. No other settings. For example there're 3 servers: a,b,c, then brpc will send requests to a, b, c, a, b, c, ... one-by-one. Note that presumption of using this algorithm is the machine specs, network latencies, server loads are similar. 

### random

Randomly choose one server from the list, no other settings. Similarly with round robin, the algorithm assumes that servers to access are similar.

### la

which is locality-aware. Perfer servers with lower latencies, until the latency is higher than others, no other settings. Check out [Locality-aware load balancing](lalb.md) for more details.

### c_murmurhash or c_md5

which is consistent hashing. Adding or removing servers does not make targets of all requests change as in simple hashing, especially suitable for caching services.

Controller.set_request_code() needs to set before RPC otherwise the RPC will fail. request_code is often the 32-bit hash code of key part of the request, and the hashing algorithm does not need to be same with the one used by load balancer. Say `c_murmurhash`  can use md5 to compute request code of the request as well.

[brpc/policy/hasher.h includes common hash functions. If `std::string key` stands for the key part of the request, controller.set_request_code(brpc::policy::MurmurHash32(key.data(), key.size())) sets request_code correctly. 

Do distinguish "the key part" and "attributes" of the request. Don't compute hash code by the full content of the request just for laziness. Minor changes in attributes may make a totally different hash code and change target server dramatically. Another cause is padding, for example: `struct Foo { int32_t a; int64_t b; }` has a 4-byte gap between `a` and `b` on 64-bit machines, which is undefined. When the request_code is computed by `hash(&foo, sizeof(foo))`,  the result is undefined. The data needs to be packed or serialized before hashing.

Check out [Consistent Hashing](consistent_hashing.md) for more details. 

## 健康检查

连接断开的server会被暂时隔离而不会被负载均衡算法选中, brpc会定期连接被隔离的server, 间隔由参数-health_check_interval控制:

| Name                      | Value | Description                              | Defined At              |
| ------------------------- | ----- | ---------------------------------------- | ----------------------- |
| health_check_interval （R） | 3     | seconds between consecutive health-checkings | src/brpc/socket_map.cpp |

一旦server被连接上, 它会恢复为可用状态. 如果在隔离过程中, server从名字服务中删除了, brpc也会停止连接尝试. 

# 发起访问

一般来说, 我们不直接调用Channel.CallMethod, 而是通过protobuf生成的桩XXX_Stub, 过程更像是"调用函数". stub内没什么成员变量, 建议在栈上创建和使用, 而不必new, 当然你也可以把stub存下来复用. Channel::CallMethod和stub访问都是**线程安全**的, 可以被所有线程同时访问. 比如: 
```c++
XXX_Stub stub(&channel);
stub.some_method(controller, request, response, done);
```
甚至
```c++
XXX_Stub(&channel).some_method(controller, request, response, done);
```
一个例外是http client. 访问http服务和protobuf没什么关系, 直接调用CallMethod即可, 除了Controller和done均为NULL, 详见[访问HTTP服务](http_client.md). 

## 同步访问

同步访问指的是: CallMethod会阻塞到server端返回response或发生错误（包括超时）. 

由于同步访问中CallMethod结束意味着RPC结束, response/controller不再会被框架使用, 它们都可以分配在栈上. 注意, 如果request/response字段特别多字节数特别大的话, 还是更适合分配在堆上. 
```c++
MyRequest request;
MyResponse response;
brpc::Controller cntl;
XXX_Stub stub(&channel);
 
request.set_foo(...);
cntl.set_timeout_ms(...);
stub.some_method(&cntl, &request, &response, NULL);
if (cntl->Failed()) {
    // RPC出错了
} else {
    // RPC成功了, response里有我们想要的回复数据. 
}
```

## 异步访问

异步访问指的是给CallMethod传递一个额外的回调对象done, CallMethod会在发出request后就结束了, 而不是在RPC结束后. 当server端返回response或发生错误（包括超时）时, done->Run()会被调用. 对RPC的后续处理应该写在done->Run()里, 而不是CallMethod后. 

由于CallMethod结束不意味着RPC结束, response/controller仍可能被框架及done->Run()使用, 它们一般得创建在堆上, 并在done->Run()中删除. 如果提前删除了它们, 那当done->Run()被调用时, 将访问到无效内存. 

你可以独立地创建这些对象, 并使用NewCallback生成done（见下文"使用NewCallback"）, 也可以把Response和Controller作为done的成员变量, 一起new出来（见下文"继承google::protobuf::Closure"）, 一般使用前一种方法. 

**发起异步请求后Request和Channel也可以立刻析构**. 
这两样和response/controller是不同的. 请注意, 这是说Channel的析构可以立刻发生在CallMethod**之后**, 并不是说析构可以和CallMethod同时发生, 删除正被另一个线程使用的Channel是未定义行为（很可能crash）. 

### 使用NewCallback
```c++
static void OnRPCDone(MyResponse* response, brpc::Controller* cntl) {
    // unique_ptr会帮助我们在return时自动删掉response/cntl, 防止忘记. gcc 3.4下的unique_ptr是public/common提供的模拟版本. 
    std::unique_ptr<MyResponse> response_guard(response);
    std::unique_ptr<brpc::Controller> cntl_guard(cntl);
    if (cntl->Failed()) {
        // RPC出错了. response里的值是未定义的, 勿用. 
    } else {
        // RPC成功了, response里有我们想要的数据. 开始RPC的后续处理.     
    }
    // NewCallback产生的Closure会在Run结束后删除自己, 不用我们做. 
}
 
MyResponse* response = new MyResponse;
brpc::Controller* cntl = new brpc::Controller;
MyService_Stub stub(&channel);
 
MyRequest request;  // you don't have to new request, even in an asynchronous call.
request.set_foo(...);
cntl->set_timeout_ms(...);
stub.some_method(cntl, &request, response, google::protobuf::NewCallback(OnRPCDone, response, cntl));
```
由于protobuf 3把NewCallback设置为私有, r32035后brpc把NewCallback独立于[src/brpc/callback.h](http://icode.baidu.com/repo/baidu/opensource/baidu-rpc/files/master/blob/src/brpc/callback.h). 如果你的程序出现NewCallback相关的编译错误, 把google::protobuf::NewCallback替换为brpc::NewCallback就行了. 

### 继承google::protobuf::Closure

使用NewCallback的缺点是要分配三次内存: response, controller, done. 如果profiler证明这儿的内存分配有瓶颈, 可以考虑自己继承Closure, 把response/controller作为成员变量, 这样可以把三次new合并为一次. 但缺点就是代码不够美观, 如果内存分配不是瓶颈, 别用这种方法. `
```c++
class OnRPCDone: public google::protobuf::Closure {
public:
    void Run() {
        // unique_ptr会帮助我们在return时自动delete this, 防止忘记. gcc 3.4下的unique_ptr是public/common提供的模拟版本. 
        std::unique_ptr<OnRPCDone> self_guard(this);
          
        if (cntl->Failed()) {
            // RPC出错了. response里的值是未定义的, 勿用. 
        } else {
            // RPC成功了, response里有我们想要的数据. 开始RPC的后续处理. 
        }
    }
 
    MyResponse response;
    brpc::Controller cntl;
}
 
OnRPCDone* done = new OnRPCDone;
MyService_Stub stub(&channel);
 
MyRequest request;  // you don't have to new request, even in an asynchronous call.
request.set_foo(...);
done->cntl.set_timeout_ms(...);
stub.some_method(&done->cntl, &request, &done->response, done);
```

### 如果异步访问中的回调函数特别复杂会有什么影响

没有特别的影响, 回调会运行在独立的bthread中, 不会阻塞其他的逻辑. 你可以在回调中做各种阻塞操作. 

### rpc发送处的代码和回调函数是在同一个线程里执行吗

一定不在同一个线程里运行, 即使该次rpc调用刚进去就失败了, 回调也会在另一个bthread中运行. 这可以在加锁进行rpc（不推荐）的代码中避免死锁. 

## 等待RPC完成
当你需要发起多个并发操作时, 可能[ParallelChannel](combo_channel.md#parallelchannel)更方便. 

如下代码发起两个异步RPC后等待它们完成. 
```c++
const brpc::CallId cid1 = controller1->call_id();
const brpc::CallId cid2 = controller2->call_id();
...
stub.method1(controller1, request1, response1, done1);
stub.method2(controller2, request2, response2, done2);
...
brpc::Join(cid1);
brpc::Join(cid2);
```
**在发起RPC前**调用Controller.call_id()获得一个id, 发起RPC调用后Join那个id. 

Join()的行为是等到**RPC结束且调用了done后**, 一些Join的性质如下: 

- 如果对应的RPC已经结束, Join将立刻返回. 
- 多个线程可以Join同一个id, RPC结束时都会醒来. 
- 同步RPC也可以在另一个线程中被Join, 但一般不会这么做.  

Join()在之前的版本叫做JoinResponse(), 如果你在编译时被提示deprecated之类的, 请修改为Join(). 

在RPC调用后Join(controller->call_id())是**错误**的行为, 一定要先把call_id保存下来. 因为RPC调用后controller可能被随时开始运行的done删除. 

下面代码的Join方式是**错误**的. 
```c++
static void on_rpc_done(Controller* controller, MyResponse* response) {
    ... Handle response ...
    delete controller;
    delete response;
}
 
Controller* controller1 = new Controller;
Controller* controller2 = new Controller;
MyResponse* response1 = new MyResponse;
MyResponse* response2 = new MyResponse;
...
stub.method1(controller1, &request1, response1, google::protobuf::NewCallback(on_rpc_done, controller1, response1));
stub.method2(controller2, &request2, response2, google::protobuf::NewCallback(on_rpc_done, controller2, response2));
...
brpc::Join(controller1->call_id());   // 错误, controller1可能被on_rpc_done删除了
brpc::Join(controller2->call_id());   // 错误, controller2可能被on_rpc_done删除了
```

## 半同步

Join可用来实现"半同步"操作: 即等待多个异步操作返回. 由于调用处的代码会等到多个RPC都结束后再醒来, 所以controller和response都可以放栈上. 
```c++
brpc::Controller cntl1;
brpc::Controller cntl2;
MyResponse response1;
MyResponse response2;
...
stub1.method1(&cntl1, &request1, &response1, brpc::DoNothing());
stub2.method2(&cntl2, &request2, &response2, brpc::DoNothing());
...
brpc::Join(cntl1.call_id());
brpc::Join(cntl2.call_id());
```
brpc::DoNothing()可获得一个什么都不干的done, 专门用于半同步访问. 它的生命周期由框架管理, 用户不用关心. 

注意在上面的代码中, 我们在RPC结束后又访问了controller.call_id(), 这是没有问题的, 因为DoNothing中并不会像上面的on_rpc_done中那样删除Controller. 

## 取消RPC

brpc::StartCancel(CallId)可取消任意RPC, CallId必须**在发起RPC前**通过Controller.call_id()获得, 其他时刻都可能有race condition. 

> 是brpc::StartCancel(CallId), 不是controller.StartCancel(), 后者被禁用, 没有效果. 

顾名思义, StartCancel调用完成后RPC并未立刻结束, 你不应该碰触Controller的任何字段或删除任何资源, 它们自然会在RPC结束时被done中对应逻辑处理. 如果你一定要在原地等到RPC结束（一般不需要）, 则可通过Join(call_id). 

关于StartCancel的一些事实: 

- call_id在发起RPC前就可以被取消, RPC会直接结束（done仍会被调用）. 
- call_id可以在另一个线程中被取消. 
- 取消一个已经取消的call_id不会有任何效果. 推论: 同一个call_id可以被多个线程同时取消, 但最多一次有效果. 
- 取消只是指client会忽略对应的RPC结果, **不意味着server端会取消对应的操作**, server cancelation是另一个功能. 

## 获取Server的地址和端口

remote_side()方法可知道request被送向了哪个server, 返回值类型是[butil::EndPoint](http://icode.baidu.com/repo/baidu/opensource/baidu-rpc/files/master/blob/src/butil/endpoint.h), 包含一个ip4地址和端口. 在RPC结束前调用这个方法都是没有意义的. 

打印方式: 
```c++
LOG(INFO) << "remote_side=" << cntl->remote_side();
printf("remote_side=%s\n", butil::endpoint2str(cntl->remote_side()).c_str());
```
## 获取Client的地址和端口

r31384后通过local_side()方法可**在RPC结束后**获得发起RPC的地址和端口. 

打印方式: 
```c++
LOG(INFO) << "local_side=" << cntl->local_side(); 
printf("local_side=%s\n", butil::endpoint2str(cntl->local_side()).c_str());
```
## 新建brpc::Controller的代价大吗

不大, 不用刻意地重用, 但Controller是个大杂烩, 可能会包含一些缓存, Reset()可以避免反复地创建这些缓存. 

在大部分场景下, 构造Controller和重置Controller(通过Reset)的代价差不多, 比如下面代码中的snippet1和snippet2性能差异不大. 
```c++
// snippet1
for (int i = 0; i < n; ++i) {
    brpc::Controller controller;
    ...
    stub.CallSomething(..., &controller);
}
 
// snippet2
brpc::Controller controller;
for (int i = 0; i < n; ++i) {
    controller.Reset();
    ...
    stub.CallSomething(..., &controller);
}
```
但如果snippet1中的Controller是new出来的, 那么snippet1就会多出"内存分配"的开销, 在一些情况下可能会慢一些. 

# 设置

Client端的设置主要由三部分组成: 

- brpc::ChannelOptions: 定义在[src/brpc/channel.h](http://icode.baidu.com/repo/baidu/opensource/baidu-rpc/files/master/blob/src/brpc/channel.h)中, 用于初始化Channel, 一旦初始化成功无法修改. 
- brpc::Controller: 定义在[src/brpc/controller.h](http://icode.baidu.com/repo/baidu/opensource/baidu-rpc/files/master/blob/src/brpc/controller.h)中, 用于在某次RPC中覆盖ChannelOptions中的选项, 可根据上下文每次均不同. 
- 全局gflags: 常用于调节一些底层代码的行为, 一般不用修改. 请自行阅读服务/flags页面中的说明. 

Controller包含了request中没有的数据和选项. server端和client端的Controller结构体是一样的, 但使用的字段可能是不同的, 你需要仔细阅读Controller中的注释, 明确哪些字段可以在server端使用, 哪些可以在client端使用. 

一个Controller对应一次RPC. 一个Controller可以在Reset()后被另一个RPC复用, 但一个Controller不能被多个RPC同时使用（不论是否在同一个线程发起）. 

Controller的特点: 
1. 一个Controller只能有一个使用者, 没有特殊说明的话, Controller中的方法默认线程不安全. 
2. 因为不能被共享, 所以一般不会用共享指针管理Controller, 如果你用共享指针了, 很可能意味着出错了. 
3. 创建于开始RPC前, 析构于RPC结束后, 常见几种模式: 
   - 同步RPC前Controller放栈上, 出作用域后自行析构. 注意异步RPC的Controller绝对不能放栈上, 否则其析构时异步调用很可能还在进行中, 从而引发未定义行为. 
   - 异步RPC前new Controller, done中删除. 
   - 异步RPC前从某个全局或thread-local的pool中取出Controller, done中Reset()并归还pool. 当然Reset()也可发生在取出时, 但在归还时能更及时地释放资源. 

## 超时

**ChannelOptions.timeout_ms**是对应Channel上一次RPC的超时, Controller.set_timeout_ms()可修改某次RPC的值. 单位毫秒, 默认值1秒, 最大值2^31（约24天）, -1表示一直等到回复或错误. 

**ChannelOptions.connect_timeout_ms**是对应Channel上一次RPC的连接超时, 单位毫秒, 默认值1秒. -1表示等到连接建立或出错, 此值被限制为不能超过timeout_ms. 注意此超时独立于TCP的连接超时, 一般来说前者小于后者, 反之则可能在connect_timeout_ms未达到前由于TCP连接超时而出错. 

注意1: brpc中的超时是deadline, 超过就意味着RPC结束. UB/hulu中的超时既有单次访问的, 也有代表deadline的. 迁移到brpc时请仔细区分. 

注意2: r31711后超时的错误码为**ERPCTIMEDOUT (1008)**, ETIMEDOUT的意思是连接超时. r31711前, 超时的错误码是ETIMEDOUT (110). 原因: RPC内很早就区分了这两者, 但考虑到linux下的使用习惯, 在RPC结束前把ERPCTIMEDOUT改为了ETIMEDOUT. 使用中我们逐渐发现不管是RPC内部实现（比如组合channel）还是一些用户场景都需要区分RPC超时和连接超时, 综合考虑后决定不再合并这两个错误. 如果你的程序中有诸如cntl->ErrorCode() == ETIMEDOUT的代码, 你考虑下这里到底是否用对了, 如果其实是在判RPC超时的话, 得改成ERPCTIMEDOUT. 

## 重试

ChannelOptions.max_retry是该Channel上所有RPC的默认最大重试次数, Controller.set_max_retry()可修改某次RPC的值, 默认值3, 0表示不重试. 

r32111后Controller.retried_count()返回重试次数. r34717后Controller.has_backup_request()获知是否发送过backup_request. 

**重试时框架会尽量避开之前尝试过的server. **

重试的触发条件有(条件之间是AND关系）: 
- 连接出错. 如果server一直没有返回, 但连接没有问题, 这种情况下不会重试. 如果你需要在一定时间后发送另一个请求, 使用backup request. 
- 没到超时. 
- 有剩余重试次数. Controller.set_max_retry(0)或ChannelOptions.max_retry = 0可关闭重试. 
- 重试对错误可能有效. 比如请求有错时(EREQUEST)不会重试, 因为server总不会接受, 没有意义. 

### 连接出错

如果server一直没有返回, 但连接没有问题, 这种情况下不会重试. 如果你需要在一定时间后发送另一个请求, 使用backup request, 工作机制如下: 如果response没有在backup_request_ms内返回, 则发送另外一个请求, 哪个先回来就取哪个. 新请求会被尽量送到不同的server. 如果backup_request_ms大于超时, 则backup request总不会被发送. backup request会消耗一次重试次数. backup request不意味着server端cancel. 

ChannelOptions.backup_request_ms影响该Channel上所有RPC, 单位毫秒, 默认值-1（表示不开启）, Controller.set_backup_request_ms()可修改某次RPC的值. 

### 没到超时

超时后RPC会尽快结束. 

### 没有超过最大重试次数

Controller.set_max_retry()或ChannelOptions.max_retry设置最大重试次数, 设为0关闭重试. 

### 错误值得重试

一些错误重试是没有意义的, 就不会重试, 比如请求有错时(EREQUEST)不会重试, 因为server总不会接受. 

r32009后用户可以通过继承[brpc::RetryPolicy](http://icode.baidu.com/repo/baidu/opensource/baidu-rpc/files/master/blob/src/brpc/retry_policy.h)自定义重试条件. r34642后通过cntl->response()可获得对应RPC的response. 对ERPCTIMEDOUT代表的RPC超时总是不重试, 即使RetryPolicy中允许. 

比如brpc默认不重试HTTP相关的错误, 而你的程序中希望在碰到HTTP_STATUS_FORBIDDEN (403)时重试, 可以这么做: 
```c++
#include <brpc/retry_policy.h>
 
class MyRetryPolicy : public brpc::RetryPolicy {
public:
    bool DoRetry(const brpc::Controller* cntl) const {
        if (cntl->ErrorCode() == brpc::EHTTP && // HTTP错误
            cntl->http_response().status_code() == brpc::HTTP_STATUS_FORBIDDEN) {
            return true;
        }
        // 把其他情况丢给框架. 
        return brpc::DefaultRetryPolicy()->DoRetry(cntl);
    }
};
...
 
// 给ChannelOptions.retry_policy赋值就行了. 
// 注意: retry_policy必须在Channel使用期间保持有效, Channel也不会删除retry_policy, 所以大部分情况下RetryPolicy都应以单例模式创建. 
brpc::ChannelOptions options;
static MyRetryPolicy g_my_retry_policy;
options.retry_policy = &g_my_retry_policy;
...
```

### 重试应当保守

由于成本的限制, 大部分线上server的冗余度是有限的, 更多是满足多机房互备的需求. 而激进的重试逻辑很容易导致众多client对server集群造成2-3倍的压力, 最终使集群雪崩: 由于server来不及处理导致队列越积越长, 使所有的请求得经过很长的排队才被处理而最终超时, 相当于服务停摆. r32009前重试整体上是安全的, 只要连接不断RPC就不会重试, 一般不会产生大量的重试请求. 而r32009后引入的RetryPolicy一方面使用户可以定制重试条件, 另一方面也可能使重试变成一场"风暴". 当你定制RetryPolicy时, 你需要仔细考虑client和server的协作关系, 并设计对应的异常测试, 以确保行为符合预期. 

## 协议

Channel的默认协议是标准协议, 可通过设置ChannelOptions.protocol换为其他协议, 这个字段既接受enum也接受字符串, 目前支持的有: 

- PROTOCOL_BAIDU_STD 或 "baidu_std", 即[标准协议](http://gollum.baidu.com/RPCSpec), 默认为单连接. 
- PROTOCOL_HULU_PBRPC 或 "hulu_pbrpc", hulu的协议, 默认为单连接. 
- PROTOCOL_NOVA_PBRPC 或 "nova_pbrpc", 网盟的协议, 默认为连接池. 
- PROTOCOL_HTTP 或 "http", http协议, 默认为连接池(Keep-Alive). 具体方法见[访问HTTP服务](http_client.md). 
- PROTOCOL_SOFA_PBRPC 或 "sofa_pbrpc", sofa-pbrpc的协议, 默认为单连接. 
- PROTOCOL_PUBLIC_PBRPC 或 "public_pbrpc", public/pbrpc的协议, 默认为连接池. 
- PROTOCOL_UBRPC_COMPACK 或 "ubrpc_compack", public/ubrpc的协议, 使用compack打包, 默认为连接池. 具体方法见[ubrpc (by protobuf)](ub_client.md). 相关的还有PROTOCOL_UBRPC_MCPACK2或ubrpc_mcpack2, 使用mcpack2打包. 
- PROTOCOL_NSHEAD_CLIENT 或 "nshead_client", 这是发送brpc-ub中所有UBXXXRequest需要的协议, 默认为连接池. 具体方法见[访问ub](ub_client.md). 
- PROTOCOL_NSHEAD 或 "nshead", 这是brpc中发送NsheadMessage需要的协议, 默认为连接池. 注意发送NsheadMessage的效果等同于发送brpc-ub中的UBRawBufferRequest, 但更加方便一点. 具体方法见[nshead+blob](ub_client.md#nshead-blob) . 
- PROTOCOL_MEMCACHE 或 "memcache", memcached的二进制协议, 默认为单连接. 具体方法见[访问memcached](memcache_client.md). 
- PROTOCOL_REDIS 或 "redis", redis 1.2后的协议（也是hiredis支持的协议）, 默认为单连接. 具体方法见[访问Redis](redis_client.md). 
- PROTOCOL_ITP 或 "itp", 凤巢的协议, 格式为nshead + control idl + user idl, 使用mcpack2pb适配, 默认为连接池. 具体方法见[访问ITP](itp.md). 
- PROTOCOL_NSHEAD_MCPACK 或 "nshead_mcpack", 顾名思义, 格式为nshead + mcpack, 使用mcpack2pb适配, 默认为连接池. 
- PROTOCOL_ESP 或 "esp", 访问使用esp协议的服务, 默认为连接池. 

## 连接方式

brpc支持以下连接方式: 

- 短连接: 每次RPC call前建立连接, 结束后关闭连接. 由于每次调用得有建立连接的开销, 这种方式一般用于偶尔发起的操作, 而不是持续发起请求的场景. 
- 连接池: 每次RPC call前取用空闲连接, 结束后归还, 一个连接上最多只有一个请求, 对一台server可能有多条连接. 各类使用nshead的协议和http 1.1都是这个方式. 
- 单连接: 进程内与一台server最多一个连接, 一个连接上可能同时有多个请求, 回复返回顺序和请求顺序不需要一致, 这是标准协议, hulu-pbrpc, sofa-pbrpc的默认选项. 

|            | 短连接                                      | 连接池                 | 单连接               |
| ---------- | ---------------------------------------- | ------------------- | ----------------- |
| 长连接        | 否 （每次都要建立tcp连接）                          | 是                   | 是                 |
| server端连接数 | qps*latency (原理见[little's law](https://en.wikipedia.org/wiki/Little%27s_law)) | qps*latency         | 1                 |
| 极限qps      | 差, 且受限于单机端口数                             | 中等                  | 高                 |
| latency    | 1.5RTT(connect) + 1RTT + 处理时间            | 1RTT + 处理时间         | 1RTT + 处理时间       |
| cpu占用      | 高每次都要tcp connect                         | 中等每个请求都要一次sys write | 低合并写出在大流量时减少cpu占用 |

框架会为协议选择默认的连接方式, 用户**一般不用修改**. 若需要, 把ChannelOptions.connection_type设为: 

- CONNECTION_TYPE_SINGLE 或 "single" 为单连接

- CONNECTION_TYPE_POOLED 或 "pooled" 为连接池, 与单个远端的最大连接数由-max_connection_pool_size控制:

  | Name                         | Value | Description                              | Defined At          |
  | ---------------------------- | ----- | ---------------------------------------- | ------------------- |
  | max_connection_pool_size (R) | 100   | maximum pooled connection count to a single endpoint | src/brpc/socket.cpp |

- CONNECTION_TYPE_SHORT 或 "short" 为短连接

- 设置为""（空字符串）则让框架选择协议对应的默认连接方式. 

r31468之后brpc支持[Streaming RPC](streaming_rpc.md), 这是一种应用层的连接, 用于传递流式数据. 

## 关闭连接池中的闲置连接

当连接池中的某个连接在-idle_timeout_second时间内没有读写, 则被视作"闲置", 会被自动关闭. 打开-log_idle_connection_close后关闭前会打印一条日志. 默认值为10秒. 此功能只对连接池(pooled)有效. 

| Name                      | Value | Description                              | Defined At              |
| ------------------------- | ----- | ---------------------------------------- | ----------------------- |
| idle_timeout_second       | 10    | Pooled connections without data transmission for so many seconds will be closed. No effect for non-positive values | src/brpc/socket_map.cpp |
| log_idle_connection_close | false | Print log when an idle connection is closed | src/brpc/socket.cpp     |

## 延迟关闭连接

多个channel可能通过引用计数引用同一个连接, 当引用某个连接的最后一个channel析构时, 该连接将被关闭. 但在一些场景中, channel在使用前才被创建, 用完立刻析构, 这时其中一些连接就会被无谓地关闭再被打开, 效果类似短连接. 

一个解决办法是用户把所有或常用的channel缓存下来, 这样自然能避免channel频繁产生和析构, 但目前brpc没有提供这样一个utility, 用户自己（正确）实现有一些工作量. 

另一个解决办法是设置全局选项-defer_close_second

| Name               | Value | Description                              | Defined At              |
| ------------------ | ----- | ---------------------------------------- | ----------------------- |
| defer_close_second | 0     | Defer close of connections for so many seconds even if the connection is not used by anyone. Close immediately for non-positive values | src/brpc/socket_map.cpp |

设置后引用计数清0时连接并不会立刻被关闭, 而是会等待这么多秒再关闭, 如果在这段时间内又有channel引用了这个连接, 它会恢复正常被使用的状态. 不管channel创建析构有多频率, 这个选项使得关闭连接的频率有上限. 这个选项的副作用是一些fd不会被及时关闭, 如果延时被误设为一个大数值, 程序占据的fd个数可能会很大. 

## 连接的缓冲区大小

-socket_recv_buffer_size设置所有连接的接收缓冲区大小, 默认-1（不修改）

-socket_send_buffer_size设置所有连接的发送缓冲区大小, 默认-1（不修改）

| Name                    | Value | Description                              | Defined At          |
| ----------------------- | ----- | ---------------------------------------- | ------------------- |
| socket_recv_buffer_size | -1    | Set the recv buffer size of socket if this value is positive | src/brpc/socket.cpp |
| socket_send_buffer_size | -1    | Set send buffer size of sockets if this value is positive | src/brpc/socket.cpp |

## log_id

通过set_log_id()可设置log_id. 这个id会被送到服务器端, 一般会被打在日志里, 从而把一次检索经过的所有服务串联起来. 不同产品线可能有不同的叫法. 一些产品线有字符串格式的"s值", 内容也是64位的16进制数, 可以转成整型后再设入log_id. 

## 附件

标准协议和hulu协议支持附件, 这段数据由用户自定义, 不经过protobuf的序列化. 站在client的角度, 设置在Controller::request_attachment()的附件会被server端收到, response_attachment()则包含了server端送回的附件. 附件不受压缩选项影响. 

在http协议中, 附件对应[message body](http://www.w3.org/Protocols/rfc2616/rfc2616-sec4.html), 比如要POST的数据就设置在request_attachment()中. 

## giano认证
```

// Create a baas::CredentialGenerator using Giano's API
baas::CredentialGenerator generator = CREATE_MOCK_PERSONAL_GENERATOR(
    "mock_user", "mock_roles", "mock_group", baas::sdk::BAAS_OK);
 
// Create a brpc::policy::GianoAuthenticator using the generator we just created 
// and then pass it into brpc::ChannelOptions
brpc::policy::GianoAuthenticator auth(&generator, NULL);
brpc::ChannelOptions option;
option.auth = &auth;
```
首先通过调用Giano API生成验证器baas::CredentialGenerator, 具体可参看[Giano快速上手手册.pdf](http://wiki.baidu.com/download/attachments/37774685/Giano%E5%BF%A
B%E9%80%9F%E4%B8%8A%E6%89%8B%E6%89%8B%E5%86%8C.pdf?version=1&modificationDate=1421990746000&api=v2). 然后按照如上代码一步步将其设置到brpc::ChannelOptions里去. 

当client设置认证后, 任何一个新连接建立后都必须首先发送一段验证信息（通过Giano认证器生成）, 才能发送后续请求. 认证成功后, 该连接上的后续请求不会再带有验证消息. 

## 重置

调用Reset方法可让Controller回到刚创建时的状态. 

别在RPC结束前重置Controller, 行为是未定义的. 

## 压缩

set_request_compress_type()设置request的压缩方式, 默认不压缩. 注意: 附件不会被压缩. HTTP body的压缩方法见[client压缩request body](http_client#压缩request-body). 

支持的压缩方法有: 

- brpc::CompressTypeSnappy : [snanpy压缩](http://google.github.io/snappy/), 压缩和解压显著快于其他压缩方法, 但压缩率最低. 
- brpc::CompressTypeGzip : [gzip压缩](http://en.wikipedia.org/wiki/Gzip), 显著慢于snappy, 但压缩率高
- brpc::CompressTypeZlib : [zlib压缩](http://en.wikipedia.org/wiki/Zlib), 比gzip快10%~20%, 压缩率略好于gzip, 但速度仍明显慢于snappy. 

下表是多种压缩算法应对重复率很高的数据时的性能, 仅供参考. 

| Compress method | Compress size(B) | Compress time(us) | Decompress time(us) | Compress throughput(MB/s) | Decompress throughput(MB/s) | Compress ratio |
| --------------- | ---------------- | ----------------- | ------------------- | ------------------------- | --------------------------- | -------------- |
| Snappy          | 128              | 0.753114          | 0.890815            | 162.0875                  | 137.0322                    | 37.50%         |
| Gzip            | 10.85185         | 1.849199          | 11.2488             | 66.01252                  | 47.66%                      |                |
| Zlib            | 10.71955         | 1.66522           | 11.38763            | 73.30581                  | 38.28%                      |                |
| Snappy          | 1024             | 1.404812          | 1.374915            | 695.1555                  | 710.2713                    | 8.79%          |
| Gzip            | 16.97748         | 3.950946          | 57.52106            | 247.1718                  | 6.64%                       |                |
| Zlib            | 15.98913         | 3.06195           | 61.07665            | 318.9348                  | 5.47%                       |                |
| Snappy          | 16384            | 8.822967          | 9.865008            | 1770.946                  | 1583.881                    | 4.96%          |
| Gzip            | 160.8642         | 43.85911          | 97.13162            | 356.2544                  | 0.78%                       |                |
| Zlib            | 147.6828         | 29.06039          | 105.8011            | 537.6734                  | 0.71%                       |                |
| Snappy          | 32768            | 16.16362          | 19.43596            | 1933.354                  | 1607.844                    | 4.82%          |
| Gzip            | 229.7803         | 82.71903          | 135.9995            | 377.7849                  | 0.54%                       |                |
| Zlib            | 240.7464         | 54.44099          | 129.8046            | 574.0161                  | 0.50%                       |                |

下表是多种压缩算法应对重复率很低的数据时的性能, 仅供参考. 

| Compress method | Compress size(B) | Compress time(us) | Decompress time(us) | Compress throughput(MB/s) | Decompress throughput(MB/s) | Compress ratio |
| --------------- | ---------------- | ----------------- | ------------------- | ------------------------- | --------------------------- | -------------- |
| Snappy          | 128              | 0.866002          | 0.718052            | 140.9584                  | 170.0021                    | 105.47%        |
| Gzip            | 15.89855         | 4.936242          | 7.678077            | 24.7294                   | 116.41%                     |                |
| Zlib            | 15.88757         | 4.793953          | 7.683384            | 25.46339                  | 107.03%                     |                |
| Snappy          | 1024             | 2.087972          | 1.06572             | 467.7087                  | 916.3403                    | 100.78%        |
| Gzip            | 32.54279         | 12.27744          | 30.00857            | 79.5412                   | 79.79%                      |                |
| Zlib            | 31.51397         | 11.2374           | 30.98824            | 86.90288                  | 78.61%                      |                |
| Snappy          | 16384            | 12.598            | 6.306592            | 1240.276                  | 2477.566                    | 100.06%        |
| Gzip            | 537.1803         | 129.7558          | 29.08707            | 120.4185                  | 75.32%                      |                |
| Zlib            | 519.5705         | 115.1463          | 30.07291            | 135.697                   | 75.24%                      |                |
| Snappy          | 32768            | 22.68531          | 12.39793            | 1377.543                  | 2520.582                    | 100.03%        |
| Gzip            | 1403.974         | 258.9239          | 22.25825            | 120.6919                  | 75.25%                      |                |
| Zlib            | 1370.201         | 230.3683          | 22.80687            | 135.6524                  | 75.21%                      |                |

# FAQ

### Q: brpc能用unix domain socket吗

不能. 因为同机socket并不走网络, 相比domain socket性能只会略微下降, 替换为domain socket意义不大. 以后可能会扩展支持. 

### Q: Fail to connect to xx.xx.xx.xx:xxxx, Connection refused是什么意思

一般是对端server没打开端口（很可能挂了）.  

### Q: 经常遇到Connection timedout(不在一个机房)

![img](../images/connection_timedout.png)

这个就是连接超时了, 调大连接和RPC超时: 

```c++
struct ChannelOptions {
    ...
    // Issue error when a connection is not established after so many
    // milliseconds. -1 means wait indefinitely.
    // Default: 200 (milliseconds)
    // Maximum: 0x7fffffff (roughly 30 days)
    int32_t connect_timeout_ms;
    
    // Max duration of RPC over this Channel. -1 means wait indefinitely.
    // Overridable by Controller.set_timeout_ms().
    // Default: 500 (milliseconds)
    // Maximum: 0x7fffffff (roughly 30 days)
    int32_t timeout_ms;
    ...
};
```

注意连接超时不是RPC超时, RPC超时打印的日志是"Reached timeout=...". 

### Q: 为什么同步方式是好的, 异步就crash了

重点检查Controller, Response和done的生命周期. 在异步访问中, RPC调用结束并不意味着RPC整个过程结束, 而是要在done被调用后才会结束. 所以这些对象不应在调用RPC后就释放, 而是要在done里面释放. 所以你一般不能把这些对象分配在栈上, 而应该使用NewCallback等方式分配在堆上. 详见[异步访问](client.md#异步访问). 

### Q: 我怎么确认server处理了我的请求

不一定能. 当response返回且成功时, 我们确认这个过程一定成功了. 当response返回且失败时, 我们确认这个过程一定失败了. 但当response没有返回时, 它可能失败, 也可能成功. 如果我们选择重试, 那一个成功的过程也可能会被再执行一次. 所以一般来说RPC服务都应当考虑[幂等](http://en.wikipedia.org/wiki/Idempotence)问题, 否则重试可能会导致多次叠加副作用而产生意向不到的结果. 比如以读为主的检索服务大都没有副作用而天然幂等, 无需特殊处理. 而像写也很多的存储服务则要在设计时就加入版本号或序列号之类的机制以拒绝已经发生的过程, 保证幂等. 

### Q: BNS中机器列表已经配置了,但是RPC报"Fail to select server, No data available"错误

使用get_instance_by_service -s your_bns_name 来检查一下所有机器的status状态,   只有status为0的机器才能被client访问.

### Q: Invalid address=`bns://group.user-persona.dumi.nj03'是什么意思
```
FATAL 04-07 20:00:03 7778 public/brpc/src/brpc/channel.cpp:123] Invalid address=`bns://group.user-persona.dumi.nj03'. You should use Init(naming_service_name, load_balancer_name, options) to access multiple servers.
```
访问bns要使用三个参数的Init, 它第二个参数是load_balancer_name, 而你这里用的是两个参数的Init, 框架当你是访问单点, 就会报这个错. 

### Q: 两个产品线都使用protobuf, 为什么不能互相访问

协议 !=protobuf. protobuf负责打包, 协议负责定字段. 打包格式相同不意味着字段可以互通. 协议中可能会包含多个protobuf包, 以及额外的长度、校验码、magic number等等. 协议的互通是通过在RPC框架内转化为统一的编程接口完成的, 而不是在protobuf层面. 从广义上来说, protobuf也可以作为打包框架使用, 生成其他序列化格式的包, 像[idl<=>protobuf](idl_protobuf.md)就是通过protobuf生成了解析idl的代码. 

### Q: 为什么C++ client/server 能够互相通信,  和其他语言的client/server 通信会报序列化失败的错误

检查一下C++ 版本是否开启了压缩 (Controller::set_compress_type), 目前 python/JAVA版的rpc框架还没有实现压缩, 互相返回会出现问题.  

# 附:Client端基本流程

![img](../images/client_side.png)

主要步骤: 

1. 创建一个[bthread_id](http://icode.baidu.com/repo/baidu/opensource/baidu-rpc/files/master/blob/src/bthread/id.h)作为本次RPC的correlation_id. 
2. 根据Channel的创建方式, 从进程级的[SocketMap](http://icode.baidu.com/repo/baidu/opensource/baidu-rpc/files/master/blob/src/brpc/socket_map.h)中或从[LoadBalancer](http://icode.baidu.com/repo/baidu/opensource/baidu-rpc/files/master/blob/src/brpc/load_balancer.h)中选择一台下游server作为本次RPC发送的目的地. 
3. 根据连接方式（单连接、连接池、短连接）, 选择一个[Socket](https://svn.baidu.com/public/trunk/baidu-rpc/src/baidu/rpc/socket.h). 
4. 如果开启验证且当前Socket没有被验证过时, 第一个请求进入验证分支, 其余请求会阻塞直到第一个包含认证信息的请求写入Socket. 这是因为server端只对第一个请求进行验证. 
5. 根据Channel的协议, 选择对应的序列化函数把request序列化至[IOBuf](http://icode.baidu.com/repo/baidu/opensource/baidu-rpc/files/master/blob/src/butil/iobuf.h). 
6. 如果配置了超时, 设置定时器. 从这个点开始要避免使用Controller对象, 因为在设定定时器后->有可能触发超时机制->调用到用户的异步回调->用户在回调中析构Controller. 
7. 发送准备阶段结束, 若上述任何步骤出错, 会调用Channel::HandleSendFailed. 
8. 将之前序列化好的IOBuf写出到Socket上, 同时传入回调Channel::HandleSocketFailed, 当连接断开、写失败等错误发生时会调用此回调. 
9. 如果是同步发送, Join correlation_id；如果是异步则至此client端返回. 
10. 网络上发消息+收消息. 
11. 收到response后, 提取出其中的correlation_id, 在O(1)时间内找到对应的Controller. 这个过程中不需要查找全局哈希表, 有良好的多核扩展性. 
12. 根据协议格式反序列化response. 
13. 调用Controller::OnRPCReturned, 其中会根据错误码判断是否需要重试. 如果是异步发送, 调用用户回调. 最后摧毁correlation_id唤醒Join着的线程. 