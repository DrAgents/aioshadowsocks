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

定义了 User 模型

| 字段        | 说明        |
| ----------- | ----------- |
| user_id     | 用户 ID     |
| port        | ss 端口     |
| method      | ss 加密方法 |
| password    | 密码        |
| enable      | 是否启用    |
| speed_limit | 限速        |

主要有两类方法 create_or_update_from_json 和 create_or_update_from_remote
分别从不同的源创建或更新用户数据.

init_user_servers 初始化用户的 ss 服务器.

## json_server

配置文件是 `./userconfigs.json`.

## remote_sync_server

会 get SS_API_ENDPOINT 获取用户数据.
会 post SS_API_ENDPOINT 刷新数据.

## grpc_server

## 参考

[sentry]: https://docs.sentry.io/error-reporting/quickstart/?platform=javascript
[go conn]: https://appliedgo.net/networking/
