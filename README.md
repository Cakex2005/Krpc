# Krpc

Krpc 是一个基于 C++11 实现的轻量级 RPC 框架。项目使用 Protobuf 描述服务与消息，使用 Muduo 处理服务端 TCP 通信，并通过 ZooKeeper 完成服务注册与发现。

当前仓库包含 RPC 核心静态库，以及一组用户登录服务示例。客户端通过 Protobuf 生成的 Stub 发起同步调用，服务端通过统一的 Provider 注册并分发 RPC 方法。

## 核心组成

- `KrpcApplication`：解析启动参数并加载全局配置。
- `KrpcProvider`：注册 Protobuf Service，接收请求并分发到对应方法。
- `KrpcChannel`：实现 `google::protobuf::RpcChannel`，完成服务发现、连接管理和同步调用。
- `Krpccontroller`：记录 RPC 调用状态与错误信息。
- `ZkClient`：封装 ZooKeeper 会话、节点创建和服务地址查询。
- `KrpcLogger`：基于 glog 提供日志输出。

## 调用链路

```text
Protobuf Stub
      │
      ▼
KrpcChannel ── ZooKeeper 服务发现
      │
      ▼ TCP
KrpcProvider ── Protobuf Service ── 业务方法
```

服务端启动时将服务写入 ZooKeeper：

```text
/<service_name>/<method_name> = <ip>:<port>
```

服务节点为持久节点，方法节点为临时节点。客户端调用方法时查询对应节点，并连接节点数据指定的 RPC 服务地址。

## 通信协议

请求帧：

```text
+------------------+-------------------+----------------+---------------+
| total_len (4 B)  | header_len (4 B)  | RPC header     | request body  |
+------------------+-------------------+----------------+---------------+
```

- `total_len`：其后所有字段的总长度，采用网络字节序。
- `header_len`：RPC Header 的序列化长度，采用网络字节序。
- `RPC header`：包含服务名、方法名和参数长度的 Protobuf 消息。
- `request body`：业务请求的 Protobuf 序列化数据。

响应帧：

```text
+---------------------+----------------+
| response_len (4 B)  | response body  |
+---------------------+----------------+
```

服务端使用 Muduo Buffer 按长度字段处理粘包与拆包；客户端按响应长度读取完整消息。

## 依赖

- Linux
- CMake 3.0+
- 支持 C++11 的编译器
- Protobuf（编译器与开发库）
- Muduo
- ZooKeeper C Client（多线程库 `zookeeper_mt`）
- glog
- pthread

Muduo 默认安装前缀为 `/root/build/release-install-cpp11`，可在配置阶段通过 `MUDUO_ROOT` 覆盖。

## 构建

```bash
mkdir -p build
cd build
cmake .. -DMUDUO_ROOT=/path/to/muduo
make -j"$(nproc)"
```

构建产物：

- `build/src/libkrpc_core.a`：RPC 核心静态库。
- `bin/server`：示例服务端。
- `bin/client`：示例客户端及并发调用程序。

## 配置与运行

程序通过 `-i` 参数读取配置文件。仓库中的 `bin/test.conf` 包含本地运行配置：

```ini
rpcserverip=127.0.0.1
rpcserverport=8000
zookeeperip=127.0.0.1
zookeeperport=2181
```

ZooKeeper 可用后，分别启动服务端与客户端：

```bash
./bin/server -i ./bin/test.conf
./bin/client -i ./bin/test.conf
```

示例服务定义在 `example/user.proto`，包含 `Login` 和 `Register` 两个 RPC 方法；当前服务端示例发布 `Login` 方法实现，客户端对该方法执行同步并发调用并输出请求统计。

## 项目结构

```text
Krpc/
├── src/                  # RPC 框架实现
│   ├── include/          # 公共头文件
│   └── Krpcheader.proto  # 框架协议头定义
├── example/
│   ├── callee/           # 示例服务端
│   ├── caller/           # 示例客户端
│   └── user.proto        # 示例服务定义
├── bin/
│   └── test.conf         # 示例运行配置
└── CMakeLists.txt
```

## 当前边界

- RPC 调用为同步阻塞模式。
- 服务通信使用 IPv4 TCP，不包含 TLS。
- 每个 RPC 方法当前注册一个服务地址，未实现负载均衡。
- `RpcController` 支持失败状态与错误文本，取消和异步回调语义尚未实现。
- 框架未提供调用超时、自动重试、熔断和鉴权机制。

## 许可证

本项目采用 [GPL-3.0](LICENSE) 许可证。
