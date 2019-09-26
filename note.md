# 学习笔记

## 环境变量与配置

- SS_GRPC_HOST
- SS_GRPC_PORT
- SS_SENTRY_DSN
- SS_API_ENDPOINT
- SS_LOG_LEVEL
- SS_SYNC_TIME
- SS_TIME_OUT_LIMIT
- SS_TCP_CONN_LIMIT

| 环境变量名        | 说明                         | 默认值      |
| ----------------- | ---------------------------- | ----------- |
| SS_GRPC_HOST      | grpc 的主机名                |             |
| SS_GRPC_PORT      | grpc 的端口                  |             |
| SS_SENTRY_DSN     | sentry 的配置 [参考][sentry] |             |
| SS_API_ENDPOINT   | api 端点                     |             |
| SS_LOG_LEVEL      | 日志级别                     | info        |
| SS_SYNC_TIME      | 同步间隔                     | 60 (单位秒) |
| SS_TIME_OUT_LIMIT | 超时时间                     | 60 (单位秒) |
| SS_TCP_CONN_LIMIT | tcp conn 的限制 (TODO: 不懂) | 60          |

如果 SS_GRPC_HOST 和 SS_GRPC_PORT 同时存在 , 启动 grpc_server

如果 SS_API_ENDPOINT 存在, 启动 remote_sync_server;
否则, 启动 json_server.

## 内存数据库

使用了 sqilte 的内存模式, 使用 peewee 作为 ORM 框架.

定义了 User 模型, 使用 User 类.

| 字段        | 说明        |
| ----------- | ----------- |
| user_id     | 用户 ID     |
| port        | ss 端口     |
| method      | ss 加密方法 |
| password    | 密码        |
| enable      | 是否启用    |
| speed_limit | 限速        |

主要方法如下:

create_or_update_from_json 和 create_or_update_from_remote
分别从不同的源创建或更新用户数据.

init_user_servers 初始化用户的 ss 服务器, 主要是调用了 UserServer 的 init_server.

---

定义了 UserServer 模型, 使用 UserServer 类.

| 字段     | 说明        |
| -------- | ----------- |
| user_id  | 用户 ID     |
| port     | ss 端口     |
| method   | ss 加密方法 |
| password | 密码        |
| enable   | 是否启用    |

类属性:

- `__running_servers__` 保存运行的服务器信息.
- `__user_limiters__` 保存用户的限制信息.
- `__user_metrics__` 保存用户的监控信息.

主要方法如下:

flush_metrics_to_remote 将监控数据发送给远程.

init_new_metric 初始化监控信息, 包含上传流量, 下载流量, tcp_conn_num 和 ip 列表.

init_server 初始化服务器, 创建 TCP 和 UDP 服务端, 并初始化限速和监控.

record_ip 记录 IP 地址.

record_traffic 更新上传和下载量.

incr_tcp_conn_num 增加 tcp_conn_num 的数量.

check_traffic_rate 检查速率, True 表示达到了限速点.

## json_server

配置文件是 `./userconfigs.json`.

## remote_sync_server

会 get SS_API_ENDPOINT 获取用户数据.
会 post SS_API_ENDPOINT 刷新数据.

会定时运行, 间隔是 SS_SYNC_TIME.

## grpc_server

使用 grpclib.server 启动了一个 grpc 服务器.

核心是 AioShadowsocksServicer.

## LocalTCP

LocalTCP 定义本地 TCP 服务器.

连接发生时, 主要调用了 LocalHandler.handle_tcp_connection_made, 同时记录了 ip 并增加 tcp_conn_num;
接收到数据时, 主要调用了 LocalHandler.handle_data_received, 同时会记录上传流量, 使用 record_traffic.

## LocalUDP

LocalUDP 定义本地 UDP 服务器.

接收到数据时, 主要调用了 LocalHandler.handle_udp_connection_made 和 LocalHandler.handle_data_received, 同时记录了上传流量, 使用了 record_traffic.

## RemoteTCP

RemoteTCP 和 LocalTCP 相对, 但多了个继承 TimeoutHandler, 用于控制连接超时.

初始化时定义了加密组件, `self.cryptor`.

`self._transport` 保存了传输负载, 也就是网络数据.

`self.local` 是一个 LocalHandler 的实例.

连接创建时, 会把 `self.data` 写入 `self._transport`

接收到数据时, 先会判断是否已经超过流量限制了, 超过了就不再处理了.
然后记录下载流量, 最后将加密过的数据写入 `self.local`.

## RemoteUDP

RemoteUDP 和 RemoteTCP 类似, 是 UDP 版的实现, 主要区分在于数据接收方法是 `datagram_received`.

## LocalHandler

LocalHandler 是绝对的核心,

> 事件循环一共处理五个状态
>
> STAGE_INIT 初始状态 socket5 握手
> STAGE_CONNECT 连接建立阶段 从本地获取 addr 进行 dns 解析
> STAGE_STREAM 建立管道(pipe) 进行 socket5 传输
> STAGE_DESTROY 结束连接状态
> STAGE_ERROR 异常状态

`self._stage` 保存事件循环的状态.

主方法是 `handle_data_received`.
首先试图解密数据, 无法解密就跳过了.
然后针对不同的事件循环状态, 跳转到对应的处理方法上.

`_handle_stage_init` 对应初始状态, 会根据协议不同, 创建 TCP 和 UDP 连接到远程网站,
也就是实际要访问的网站. RemoteTCP 和 RemoteUDP 的初始化就在这里.
TCP 会先进入一个状态, STAGE_CONNECT 即连接状态. UDP 无需这个状态.
成功以后, 都会进入 STAGE_STREAM 状态,

`_handle_stage_stream` 用于传输数据, 直接将解密后的数据发送给远程网站就行了.

## 加解密

所有传输的数据都要经过加密, 从 client 接收到的数据要经过解密后传输给 remote (即目标网站), 从目标网站获得的数据, 要经过加密后传输给 client.

```
  加密   -->         解密      -->
client             代理服务器         目标网站
  解密   <--         加密      <--
```

当前支持的加密方法

- aes 系列: {"aes-128-cfb": 16, "aes-192-cfb": 24, "aes-256-cfb": 32}
- none: 不加密

## 参考

[sentry]: https://docs.sentry.io/error-reporting/quickstart/?platform=javascript
[go conn]: https://appliedgo.net/networking/
