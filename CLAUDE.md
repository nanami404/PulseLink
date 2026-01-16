# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

PulseLink 是一个基于事件驱动架构的心跳检测与 WebSocket 实时连接系统，主要解决客户端连接不稳定、设备状态错乱和第三方系统无法获取实时数据的问题。

**核心架构特点：**
- **事件驱动**：使用 Redis Pub/Sub 完全取代定时任务和任务队列
- **状态集中**：Redis 作为单一真相源存储所有状态（带自动过期机制）
- **多实例部署**：通过 Nginx 负载均衡（least_conn 策略）支持横向扩展
- **跨节点通信**：实例间通过 Redis Pub/Sub 事件广播协调

**技术栈：**
- Spring Boot 3.x（JDK 17+）
- Netty Server（WebSocket）
- Redis 6.0+（Pub/Sub + Key 过期机制）
- Nginx 1.18+（WSS 代理）

## 核心概念

### Redis 数据结构设计

系统依赖以下 Redis 数据结构进行状态管理和通信：

**Key 模式：**
- `devices:online` (Set) - 在线设备列表，存在即表示 online
- `device:heartbeat:{deviceId}` (String, TTL 120s) - 最新心跳时间戳，过期自动触发离线
- `device:runtimeInfo:{deviceId}` (String, TTL 300s) - 运行数据 JSON 字符串
- `nodes:online` (Set) - 在线实例 UUID 列表
- `node:heartbeat:{nodeId}` (String, TTL 120s) - 实例心跳时间戳
- `lock:node:master` (String, TTL 30s) - 分布式锁，主节点选举用

**Pub/Sub Channel：**
- `client:status:change` - 设备上线/离线事件
- `client:heartbeat` - 心跳事件（可选，用于监控）
- `client:data:request` - 数据请求事件
- `client:data:updated` - 数据更新事件

### WebSocket 消息格式

所有 WebSocket 消息使用统一格式：

```json
{
  "mapper": "消息类型",
  "body": { /* 具体数据 */ }
}
```

**mapper 类型：**
- `register` - 客户端注册
- `ping` / `pong` - 心跳
- `getRuntimeInfo` - 服务器请求运行数据
- `runtimeInfo` - 客户端上报运行数据

### HTTP API 设计

提供统一的 `ApiResult<T>` 响应格式：

```json
{
  "code": 200,
  "message": "提示信息",
  "data": { /* 数据体 */ }
}
```

**关键 API：**
- `GET /api/client/{deviceId}/runtimeinfo` - 查询运行数据（先读缓存，缓存过期则通过 Pub/Sub 请求客户端）

## 关键功能模块

### 1. 客户端连接管理

**注册流程（F002）：**
1. 验证 JSON 格式和 deviceId
2. 将 deviceId 加入 `devices:online` Set
3. 初始化 `device:heartbeat:{deviceId}`（TTL 120s）
4. 发布 `client:status:change` 事件（status=online）
5. 异步 POST 通知第三方系统（含重试机制）

**心跳机制（F003-F004）：**
- 客户端每 30s 发送 ping
- 服务器更新 `device:heartbeat:{deviceId}`（重置 TTL）
- Redis Key 过期（120s 无心跳）自动触发离线：
  - 从 `devices:online` 移除
  - 发布 `client:status:change` 事件（status=offline）
  - POST 通知第三方

**断线处理（F005）：**
- 正常关闭（close 帧）：立即清理 + 通知
- 异常掉线：等待心跳超时（由 Redis Key 过期处理）

### 2. 运行数据查询（F006-F007）

**查询流程：**
1. 检查 Redis 缓存 `device:runtimeInfo:{deviceId}`
2. 缓存有效（300s 内）：直接返回
3. 缓存无效或设备离线：返回失败
4. 缓存无效但设备在线：
   - 生成 requestId
   - 发布 `client:data:request` 事件到 Pub/Sub
   - 持有该客户端连接的实例收到事件
   - 通过 Netty Channel 发送 `getRuntimeInfo` 消息给客户端
   - 客户端响应后更新 Redis 缓存
   - 发布 `client:data:updated` 事件
   - HTTP 请求从 Redis 获取数据并返回（最多等待 5 秒）

### 3. 多实例节点管理（F010-F012）

**节点注册：**
- Spring Boot 启动时生成 UUID
- 加入 `nodes:online` Set
- 初始化 `node:heartbeat:{nodeId}`（TTL 120s）

**节点心跳：**
- 定时任务每 20s 更新一次心跳

**节点超时检测：**
- 通过 Redis 分布式锁（`lock:node:master`）选举主节点
- 主节点每 60s 扫描一次，清理超时节点
- 从 `nodes:online` 移除超时节点

### 4. 第三方通知（F008-F009）

**重试机制（指数退避）：**
- 第 1 次：立即重试
- 第 2 次：等待 1s
- 第 3 次：等待 2s
- 第 4 次：等待 4s
- 仍失败：记录 ERROR 日志

**通知地址：** `POST https://ip/device/api/updateDeviceStatus`

**请求体：**
```json
{
  "deviceId": "device-001",
  "status": "online" // or "offline"
}
```

## 时间参数配置

| 参数 | 值 | 说明 |
|------|-----|------|
| 客户端心跳间隔 | 30 秒 | 客户端定时发送 ping |
| 心跳超时时间 | 120 秒 | Redis String TTL，4 倍心跳间隔 |
| 运行数据缓存 | 300 秒 | Redis TTL |
| 节点心跳间隔 | 20 秒 | 实例定时更新心跳 |
| 节点心跳超时 | 120 秒 | Redis String TTL |
| 主节点扫描间隔 | 60 秒 | 扫描离线节点 |
| 第三方通知超时 | 10 秒 | 单次 HTTP 请求超时 |
| 第三方重试次数 | 3 次 | 指数退避：1s → 2s → 4s |

## 日志要求

**日志级别：**
- **DEBUG**：详细流程（心跳更新、Pub/Sub 事件）
- **INFO**：正常流程（连接建立、注册成功、通知成功）
- **WARN**：警告信息（异常掉线、节点超时）
- **ERROR**：失败和异常（JSON 解析失败、通知失败）

**必须记录的操作：**
- 客户端连接建立/断开
- 注册成功/失败
- 心跳接收（DEBUG）
- 上/下线事件
- 第三方通知成功/失败
- JSON 解析异常
- 运行数据更新（DEBUG）
- 节点启动/心跳/超时

**日志框架：** SLF4J + Logback

## 异常处理原则

1. **JSON 解析失败**：不关闭连接，返回错误响应，记录 ERROR 日志
2. **第三方通知失败**：指数退避重试 3 次，失败后记录 ERROR 日志
3. **客户端无响应**：5 秒超时，返回失败
4. **Redis 连接断开**：记录 ERROR，拒绝新连接
5. **异常掉线**：等待心跳超时后清理（不立即触发离线事件）

## 开发阶段

按以下优先级开发：

**第一阶段：**
- F001-F002：客户端连接与注册
- F003-F005：心跳检测与离线判断
- Redis 数据结构和 Pub/Sub 基础设施

**第二阶段：**
- F006-F007：运行数据查询与上报
- 跨节点通信（Pub/Sub 请求/响应）

**第三阶段：**
- F008-F009：第三方状态通知（含重试机制）
- F010-F012：多实例节点管理

**第四阶段：**
- F013-F014：日志与异常处理
- 性能测试与调优

## 性能指标

| 指标 | 要求 |
|------|------|
| 并发连接数 | 单实例支持 1000+ 客户端 |
| 心跳响应时间 | < 1 秒 |
| Pub/Sub 延迟 | < 100 毫秒 |
| 运行数据查询 | < 5 秒 |
| HTTP API 响应（缓存命中） | < 500 毫秒 |
| 系统可用率 | ≥ 99.5% |
| 故障恢复时间 | < 30 秒 |

## 部署架构

```
第三方系统
    │ HTTP POST
    ▼
Nginx (WSS 443, least_conn)
    │
    ├──▶ Instance 1 ──┐
    ├──▶ Instance 2 ──┤
    └──▶ Instance 3 ──┘
                      │
          ┌───────────▼─────────────┐
          │      Redis Server       │
          │  (Pub/Sub + 状态存储)   │
          └─────────────────────────┘

客户端设备群 (WSS 连接 → 自动重连 → 心跳 30s)
```

## 注意事项

1. **无状态设计**：每个实例只管理自己连接的客户端，状态全在 Redis
2. **查询任意化**：任意实例都能处理 HTTP 请求（通过 Redis + Pub/Sub 转发）
3. **过期驱动**：大量使用 Redis Key 过期机制触发离线检测
4. **异步通知**：第三方通知必须异步处理，不能阻塞主流程
5. **幂等性**：注册、心跳、离线等操作要支持重复调用
6. **连接本地化**：只有持有客户端 Netty Channel 的实例才能直接发送消息给该客户端
