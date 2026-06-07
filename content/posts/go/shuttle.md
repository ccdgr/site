---
date: '2025-03-15T10:30:00+08:00'
draft: false
title: '校车预定平台演进：从 MySQL 裸写到 Redis + RabbitMQ 高并发架构'
tags: ["Go", "MySQL", "Redis", "RabbitMQ"]
description: "一次压力测试驱动的架构演进：网络延迟的发现、中间件的引入决策、6 个版本的性能对比"
toc: false
---

> 做这个项目的初衷不是"写一个能用的系统"，而是搞清楚一件事：中间件到底在解决什么问题？每一个组件引入之前，先把不引入的代价量化出来。所以才有了一版一版的刻意劣化测试。

---

## 项目背景

独立开发的西工大校车预定平台，技术栈 Go + Gin + Gorm + MySQL，后来逐步引入 Redis、RabbitMQ。核心业务是座位预约、库存扣减与订单管理。

面向的并发场景：热门班次开放预约的瞬间，几千人同时抢几十个座位。

我的思路是：**先写一个"能跑但没做任何优化"的版本，压测，找到瓶颈，加一个组件，再压测，直到瓶颈消失。** 每一步都带数据。

---

## 前置准备：基准环境与测试工具

```
硬件：Apple M4
MySQL：Docker 1核1G，单实例
Redis：Docker 单实例
JMeter：threads=2000, ramp-up=1s, loop=1
网络延迟模拟：tc qdisc add dev lo0 root netem delay 5ms
```

测试接口：`POST /api/order`，包含查库存、扣库存、插入订单三步操作。

---

## 为什么需要模拟网络延迟

第一个压测在本地回环上跑，MySQL 1核1G，2000 并发，结果 TPS 400+，0 错误。

但这是假象。**生产环境中应用服务器到数据库服务器至少 2-5ms 的网络 RTT。** 本地走 `localhost` 延迟为 0，意味着每个 SQL 操作节省了 5-10ms。一个请求有 3 次 SQL（查库存 + 更新 + 插入），本地比生产少了至少 15ms 的等待。2000 并发时，15ms × 2000 ÷ 1s ≈ 30 个并发一直在"多出来"的等待里打转。

```bash
# Apple M4 macOS 上模拟机房 5ms 延迟
sudo dnctl pipe 1 config delay 5
sudo pfctl -E
```

加上 5ms 延迟后，同样的代码、同样的数据，TPS 从 400 直接掉到 20。

**以下所有测试默认开启 5ms 网络延迟。** 我要测的是：在这 20 TPS 的基础上，每一步优化能带来多少提升，以及为什么。

---

## v1：有缺陷的基准线

```go
func HandleOrder(c *gin.Context) {
    var req OrderReq
    c.ShouldBindJSON(&req)

    tx := db.Begin()
    var stock Stock
    tx.Where("seat_id = ?", req.SeatID).First(&stock)
    if stock.Count <= 0 {
        tx.Rollback()
        return
    }
    tx.Model(&stock).Update("count", stock.Count-1)
    tx.Create(&Order{SeatID: req.SeatID, UserID: req.UserID})
    tx.Commit()
}
```

这个版本**事务、库存校验都在**，逻辑上没有硬伤。唯一的缺陷：没有超时控制。

| 指标 | 值 |
|---|---|
| MAX 响应 | 99,285 ms |
| Error | 0.0% |
| TPS | 20.1/sec |

0 错误，但请求全堵死了。原因：MySQL 连接池有限，2000 个 goroutine 同时在等数据库返回。前面的请求没完成，后面的请求排不上队。最慢的一个等了 99 秒。

**v1 说明什么：不是事务写得不对，是缺少超时控制导致连接被耗尽。Gin 默认不设超时，goroutine 会一直等到 MySQL 返回或 TCP 超时（默认几分钟）。在延迟环境下，这直接让系统不可用。**

---

## v2：加入超时控制——"快速失败"

加 Context 超时，没什么技巧：

```go
ctx, cancel := context.WithTimeout(c.Request.Context(), 3*time.Second)
defer cancel()
tx := db.WithContext(ctx).Begin()
// ...
```

| 指标 | 值 |
|---|---|
| MAX 响应 | 3,015 ms |
| Error | 95.50% |
| TPS | 399.8/sec（名义）|

TPS 从 20 → 400？只是因为超时后快速返回错误。**真正成功的 TPS 还是 20。** 95.5% 的请求都超时失败了。

现在看到的"吞吐提升"，实际上是把之前排队阻塞的请求变成了即时失败。这一步的价值在于告诉用户"系统忙，请稍后"，而不是让用户干等 99 秒。但问题本身没有解决：为什么只有 20 个请求能成功？

**v2 说明什么：超时只是手段，瓶颈在 MySQL 连接能力本身。**

---

## v3：优化事务——缩短持锁时间

v2 的分析显示瓶颈在 MySQL。一个自然的想法：能不能让事务更快？

```go
// 之前：事务中做 SELECT + UPDATE + INSERT，三步串行
// 优化：合并查询条件，减少一次 SELECT
tx.Model(&Stock{}).
    Where("seat_id = ? AND count > 0", req.SeatID).
    Update("count", gorm.Expr("count - 1"))
```

| 指标 | 值 |
|---|---|
| MAX 响应 | 2,518 ms |
| Error | 90.05% |
| 实际 TPS | 50/sec |

实际成功 TPS 从 20 → 50。原因是每个事务持锁时间更短，单位时间内能完成更多事务。但 90% 的错误率说明 MySQL 仍然是瓶颈，10% 的成功率约等于 "2000 人里面 200 人抢到票"，体验仍然很差。

另外发现了一个问题：**订单数与库存扣减不一致。** 高并发下 `SELECT ... WHERE count > 0` 和 `UPDATE count - 1` 之间有竞态窗口。这是 MySQL 行级锁也解决不了的吗？实际上行锁只保证同一行不会同时被写，但两个不同事务可以同时读到 count=1，都认为有库存，都执行 UPDATE，最终 count 变成 -1。

---

## v4：调整执行顺序——先扣库存再写订单

倒过来执行，扣库存在前：

```go
// 先写库存（行级锁立即获取），再写订单
tx.Model(&Stock{}).Where("seat_id = ? AND count > 0", req.SeatID).
    Update("count", gorm.Expr("count - 1"))
tx.Create(&Order{...})
```

| 指标 | 值 |
|---|---|
| MAX 响应 | 2,518 ms |
| Error | 90.05% |
| 实际 TPS | 50/sec |

没有改善。**原因：调整事务内部的语句顺序，不改变事务总时长和锁竞争格局。** 瓶颈不在 SQL 的执行顺序，而在数据库连接数有限 + 网络延迟让每个连接被占用的时间被拉长。

**v4 说明什么：SQL 层面的优化已经到天花板了。要想 TPS 再上一个量级，必须把一部分读/写负载从 MySQL 转移出去。**

---

## v5：引入 Redis 做库存缓存——方向对了，实现错了

思路：把"查库存"这个最频繁的操作从 MySQL 搬到 Redis。Redis 单线程 + 内存操作，不受网络延迟以同样程度的影响（一次 Redis 命令是一次网络往返，不需要等事务提交）。

```go
// 查库存：走 Redis
count := redis.Get(ctx, "stock:seat:"+req.SeatID).Int()
if count <= 0 {
    return errors.New("sold out")
}
// 扣库存：走 Redis
redis.Decr(ctx, "stock:seat:"+req.SeatID)
// 写订单：走 MySQL
db.WithContext(ctx).Create(&order)
```

| 指标 | 值 |
|---|---|
| MAX 响应 | 108 ms |
| Error | 87.05% |
| TPS | 996.5/sec |

TPS 从 50 → 997，响应时间从 3s → 108ms。**这一步证明了：把读负载从 MySQL 转移到 Redis 是对的。**

但 v5 有两个致命缺陷：

1. **`GET` + `DECR` 不是原子操作。** 两个 goroutine 同时 GET 到 count=1，都判断还有库存，都执行 DECR，库存变成 -1
2. **大量请求绕过 Redis 后涌入 MySQL。** 997 TPS 的请求虽然通过 Redis 过了库存校验，但在实际进行扣减和写入数据库的时候并没有减少连接占用，87% 的错误率说明 MySQL 仍然是瓶颈

**v5 说明什么：方向对了（Redis 缓存可以大幅降低延迟），但需要原子性保障 + 削峰手段。**

---

## v6：Redis + Lua 原子扣库存

```lua
-- deduct_stock.lua
local key = KEYS[1]
local stock = redis.call('GET', key)
if tonumber(stock) <= 0 then
    return -1  -- 已售罄
end
redis.call('DECR', key)
return stock - 1
```

```go
result := redis.Eval(ctx, luaScript, []string{"stock:seat:" + req.SeatID}).Int()
```

| 指标 | 值 |
|---|---|
| MAX 响应 | 23 ms |
| Error | 90.05% |
| TPS | 996.5/sec |

库存准确性✅，超卖完全解决。Lua 脚本在 Redis 服务端原子执行，单线程模型天然避免了竞态。

但 Error 仍然是 90%。**原因：库存扣减正确了，但 2000 个请求中可能有 1000 个通过了 Redis 校验（假设初始库存 500，但每个请求只扣 1 个座位），500 个合法请求 + 500 个排队请求同一瞬间进入 MySQL，数据库撑不住。**

**v6 说明什么：原子性解决了数据正确性问题，但并发写 MySQL 的瓶颈依然存在。下一个要解决的是：减小 MySQL 写入的峰值。**

---

## 六轮压测总览

| 版本 | 核心改动 | MAX | Error | 实际 TPS | 量化了什么问题 |
|---|---|---|---|---|---|
| v1 | 基础实现，无超时 | 99s | 0% | 20 | 无超时 → 连接耗尽 |
| v2 | +Context 超时 | 3s | 95.5% | 20 | 超时只是兜底，不提升真实吞吐 |
| v3 | +精简事务 SQL | 2.5s | 90% | 50 | SQL 优化有上限，延迟环境放大瓶颈 |
| v4 | +调整执行顺序 | 2.5s | 90% | 50 | 锁竞争本身不是问题，问题是持有连接的时间 |
| v5 | +Redis 缓存库存 | 108ms | 87% | 997 | 读负载卸载有效，但非原子操作导致超卖 |
| v6 | +Redis+Lua 原子 | 23ms | 90% | 997 | 原子性解决正确性，但写高峰仍在 MySQL |

---

## v7：最终架构 — RabbitMQ 削峰 + 可靠投递

v6 之后，库存扣减的响应时间已经优化到 23ms，瓶颈转移到了 MySQL 写入。解决方案很明确：**不要让 2000 个请求同时写 MySQL。**

```
客户端 → Gin API → Redis+Lua 扣库存 → RabbitMQ → 消费者批量写 MySQL
```

```go
// API 层：扣库存 + 发消息，快速返回
func HandleOrder(c *gin.Context) {
    result := redis.Eval(ctx, luaDeduct, []string{key}).Int()
    if result == -1 {
        return errors.New("sold out")
    }
    ch.PublishWithContext(ctx, "", "order.queue", false, false, amqp.Publishing{
        Body: msg.Bytes(),
    })
}

// 消费者：批量攒批 + 幂等写入
func consumer() {
    batch := make([]Order, 0, 50)
    ticker := time.NewTicker(200 * time.Millisecond)
    for {
        select {
        case msg := <-msgs:
            batch = append(batch, parse(msg))
            if len(batch) >= 50 { flush(batch); batch = batch[:0] }
        case <-ticker.C:
            flush(batch); batch = batch[:0]
        }
    }
}
```

可靠性保证：

- **生产者确认**：`PublisherConfirm` 模式，消息未到达 Broker 则回滚 Redis 库存
- **幂等消费**：订单表 `order_id` 加唯一约束，重复消息 `INSERT IGNORE`
- **死信队列**：消费失败的消息投递到 DLQ，定时巡检补单

最终吞吐量提升 2 倍，库存零超卖。

---

## 关键结论

每个中间件解决的是特定约束下的特定问题：

| 组件 | 解决的问题 | 如果没有它 |
|---|---|---|
| Context 超时 | 请求不响应 → 连接耗尽 | goroutine 永远等待 |
| Redis | MySQL 连接数有限 × 网络延迟放大 | 每个请求等 10ms × N 个请求 |
| Lua | 读-判断-写 不是原子的 | 超卖 |
| RabbitMQ | 瞬时高并发写 | MySQL 被写请求排队打满 |

**最重要的发现：本地压测一定一定一定要加网络延迟。** 如果不加那 5ms，v1 就能跑到 400 TPS，后面的所有探索都不会发生。生产环境的网络延迟是把所有隐藏问题放大出来的放大镜。

这篇文章的目的：**用可控的变量、可复现的压测数据，证明每个中间件引入的必要性，而不是凭直觉说"高并发就该用 Redis"。**
