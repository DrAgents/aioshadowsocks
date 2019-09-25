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

* `__running_servers__` 保存运行的服务器信息.
* `__user_limiters__` 保存用户的限制信息.
* `__user_metrics__` 保存用户的监控信息.

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
接收到数据时, 主要调用了 LocalHandler.handle_data_received, 同时会记录流量, 使用 record_traffic.

## LocalUDP

LocalUDP 定义本地 UDP 服务器.


## 参考

[sentry]: https://docs.sentry.io/error-reporting/quickstart/?platform=javascript
[go conn]: https://appliedgo.net/networking/
