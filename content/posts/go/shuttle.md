---
date: '2025-03-15T10:30:00+08:00'
draft: false
title: '校车预定平台演进：从 MySQL 裸写到 Redis + RabbitMQ 高并发架构'
tags: ["Go", "MySQL", "Redis", "RabbitMQ"]
description: "一次压力测试驱动的架构演进：网络延迟的发现、中间件的引入决策、6 个版本的性能对比"
toc: false
---

> 本地测 MySQL 轻松扛住 2000 并发，模拟 5ms 延迟后直接崩了。中间件到底有没有用？用数据说话。

---

## 项目背景

独立开发的西工大校车预定平台，技术栈 Go + Gin + Gorm + MySQL + Redis + RabbitMQ + Vue。核心业务是座位的预约、扣减与订单管理，属于典型的库存扣减场景。

最开始我的架构极其简单：**客户端 → Gin API → MySQL**。CRUD 而已，能跑。

---

## 第一次压力测试：MySQL 完全没问题？

写完第一版后打了个 JMeter 压测：

```
环境：Apple M4, MySQL 1核1G
JMeter: threads=2000, ramp-up=1s, loop=1
网络：本地回环（0ms 延迟）

结果：TPS 400+，Error 0%，MySQL 毫无压力。
```

我一度怀疑这个项目太简单了，根本不需要什么 Redis、消息队列。

## 转折：本地 MySQL 没有网络延迟

后来注意到一个细节：机房里的应用服务器和数据库服务器之间**至少有 2-5ms 的网络延迟**。而我本地压测走的是 `localhost`，网络延迟为 0。

用 Linux `tc` 模拟 5ms 延迟后重新压测，结果天差地别：

```bash
# 模拟 5ms 网络延迟
tc qdisc add dev lo0 root netem delay 5ms
```

之前 400 TPS 的 MySQL，加上 5ms 延迟后直接掉到 20 TPS。

> 为什么？每次请求有 3-4 次数据库操作（查库存、插订单、减库存），每次都得多等 5ms × 2（往返）= 10ms。2000 个并发请求排队等 MySQL 连接，max 响应时间飙升到 99 秒。

**这是整篇文章最重要的结论之一：本地压测不模拟网络延迟，结果毫无参考价值。**

---

## 演进 v1 → v6：引入中间件到底带来了什么

以下所有测试均在 5ms 网络延迟下进行。

### v1：裸写 MySQL，无超时控制

```go
func HandleOrder(c *gin.Context) {
    // 查库存
    db.First(&stock, "id = ?", seatID)
    // 扣库存
    db.Model(&stock).Update("count", gorm.Expr("count - 1"))
    // 插入订单
    db.Create(&order)
}
```

| 指标 | 值 |
|---|---|
| MAX 响应 | 99,285 ms（99 秒！） |
| Error | 0% |
| TPS | 20.1/sec |

0 错误但请求全部堵死，因为没有超时控制，goroutine 一直等 MySQL 连接。

### v2：引入 Context 超时控制

```go
ctx, cancel := context.WithTimeout(c.Request.Context(), 3*time.Second)
defer cancel()
db.WithContext(ctx).Create(&order)
```

| 指标 | 值 |
|---|---|
| MAX 响应 | 3,015 ms |
| Error | 95.50% |
| TPS | 399.8/sec |

TPS 看起来从 20 → 400，大幅提升？**实际上真实成功的 TPS 还是 20**，只是超时后快速返回错误，不再占用连接等待。大量请求超时失败，95% 的错误率意味着只有 5% 的用户能正常下单。

### v3：简化事务中的 SQL

```go
// 之前：事务中先后执行 UPDATE + INSERT + UPDATE
// 之后：合并为一条 UPDATE，减少事务持锁时间
```

| 指标 | 值 |
|---|---|
| MAX 响应 | 3,015 ms |
| Error | 90.05% |
| TPS | 400.0/sec |

实际 TPS 从 20 提升到 50，因为事务更短、数据库超时更少。但订单数量与库存对不上 — 出现了超卖。

### v4：先扣库存再插订单

调整执行顺序，库存扣减先于订单插入，试图减少锁冲突。

| 指标 | 值 |
|---|---|
| MAX 响应 | 3,015 ms |
| Error | 90.05% |
| TPS | 50/sec（实际） |

没有实质性改善。**瓶颈在于 MySQL 连接被 5ms 延迟放大，单纯调 SQL 顺序解决不了问题。**

### v5：引入 Redis 缓存库存

```go
// 查库存走 Redis
count, _ := redis.Get(ctx, "stock:seat:1").Int()
if count <= 0 {
    return errors.New("sold out")
}
// 扣库存（有竞态！）
redis.Decr(ctx, "stock:seat:1")
// 写 MySQL
db.Create(&order)
```

| 指标 | 值 |
|---|---|
| MAX 响应 | 108 ms |
| Error | 87.05% |
| TPS | 996.5/sec |

TPS 从 50 → 1000，响应时间从 3 秒 → 108ms。Redis 的单线程模型和内存操作使得库存查询不再受 MySQL 连接数和网络延迟的限制。

**但是问题很大：**

1. Redis `GET` + `DECR` 不是原子操作，仍会出现超卖
2. 经过 Redis 拦截的合法请求同一时刻涌入 MySQL，导致大量 SQL 超时（87% 的错误率几乎没降）
3. 订单数据不可靠 — 库存扣了但订单可能没插成功

### v6：Redis + Lua 原子扣减

```lua
-- stock_deduct.lua
local stock = redis.call('GET', KEYS[1])
if tonumber(stock) <= 0 then
    return 0
end
return redis.call('DECR', KEYS[1])
```

```go
result, _ := redis.Eval(ctx, luaScript, []string{"stock:seat:1"}).Int()
```

| 指标 | 值 |
|---|---|
| MAX 响应 | 23 ms |
| Error | 90% |
| TPS | 996.5/sec |

库存准确性✅，不再超卖。但 Error 还是 90% — 因为分布式锁或者队列加速机制并没有引入，大量的请求阻塞在了 MySQL 写入这一步。

---

## 测试数据一览

| 版本 | 改动 | MAX | Error | TPS | 问题 |
|---|---|---|---|---|---|
| v1 | 裸 MySQL | 99s | 0% | 20 | 请求堵死 |
| v2 | +Context 超时 | 3s | 95.5% | 20(实际) | 大量超时失败 |
| v3 | +简化事务 SQL | 3s | 90% | 50(实际) | 超卖 |
| v4 | +先扣库存 | 3s | 90% | 50(实际) | 无改善 |
| v5 | +Redis 缓存 | 108ms | 87% | 997 | 数据不可靠 |
| v6 | +Redis+Lua 原子 | 23ms | 90% | 997 | 写入瓶颈 |

---

## v7：最终架构 — RabbitMQ 异步削峰

v6 的瓶颈已经从库存查扣转移到了 MySQL 写入。解决方案是引入消息队列：

```
客户端 → Gin API → Redis+Lua 扣库存（同步）→ RabbitMQ 发消息 → 消费者批量写 MySQL
```

```go
// API 层：扣库存后发消息
func HandleOrder(c *gin.Context) {
    // 1. Redis+Lua 原子扣库存
    ok := stock.Deduct(ctx, seatID, 1)
    if !ok {
        return errors.New("sold out")
    }
    // 2. 发消息到 RabbitMQ，快速返回
    msg := OrderMsg{SeatID: seatID, UserID: userID}
    ch.PublishWithContext(ctx, "", "order.queue", false, false, amqp.Publishing{
        ContentType: "application/json",
        Body: msg.Marshal(),
    })
    return nil // 用户即刻得到响应
}

// 消费者：批量写入 MySQL
func OrderConsumer(msgs <-chan amqp.Delivery) {
    var batch []Order
    ticker := time.NewTicker(200 * time.Millisecond)
    for {
        select {
        case msg := <-msgs:
            batch = append(batch, parseOrder(msg))
            if len(batch) >= 50 {
                flushBatch(batch)
                batch = nil
            }
        case <-ticker.C:
            flushBatch(batch)
            batch = nil
        }
    }
}
```

完整性保障：

- **生产者确认**：`ch.PublishWithContext` + `Confirm` 模式，消息未到达 Broker 则回滚库存
- **消费者幂等**：订单表 `order_id` 加唯一索引，重复消费直接 `INSERT IGNORE`
- **死信队列**：消费失败的消息进入 DLQ，定时巡检补单

最终架构下核心接口吞吐量提升 2 倍，库存零超卖。

---

## 核心结论

回到最初的问题：**中间件到底有没有用？**

| 中间件 | 解决的问题 | 带来的提升 |
|---|---|---|
| Redis | 热数据读压力 + 原子库存扣减 | 响应时间 99s→23ms，TPS 20→997 |
| RabbitMQ | 同步写入瓶颈 + 流量削峰 | 吞吐量再翻倍，写入不丢消息 |
| Context | 无超时导致的连接泄漏 | 请求不再阻塞，快速失败 |
| Lua | Redis 读写竞态 | 库存零超卖 |

**最重要的经验：本地压测一定要模拟网络延迟。** 如果不加那 5ms，我永远不知道 MySQL 在真实环境下有多脆弱，也永远无法做出引入 Redis 的正确决策。

---

## 思考

引进中间件的同时也引入了复杂性，比如 RocketMQ/Redis 挂了怎么办，这些设计到异地多活和降级，并不是本文讨论的范围。

一个可用的系统，总是伴随着大量的妥协和权衡，这篇文章的目的很简单：**用数据证明，在某些场景下，中间件确实能带来数量级上的性能提升。**
