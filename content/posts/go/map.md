---
date: '2026-06-07T15:27:05+08:00'
draft: false
title: 'Map'
---

# Go 并发安全 Map 三种实现对比：从互斥锁到分段锁

> 实验环境：Go 1.24, Apple M4, macOS 15, 10 核 CPU
>
> 所有测试均通过 `go test -race` 竞态检测，零数据竞争。

---

## 目录

1. [问题背景](#问题背景)
2. [解法一：sync.RWMutex](#解法一syncrwmutex)
3. [解法二：sync.Map](#解法二syncmap)
4. [解法三：分段锁 ShardMap](#解法三分段锁-shardmap)
5. [测试策略与用例](#测试策略与用例)
6. [Benchmark 性能对比](#benchmark-性能对比)
7. [分片均匀性验证](#分片均匀性验证)
8. [竞态检测](#竞态检测)
9. [如何选择](#如何选择)
10. [完整源码](#完整源码)

---

## 问题背景

Go 原生的 `map` **不是并发安全的**。多个 goroutine 同时读写同一个 map 会触发 race condition，轻则数据错乱，重则 `fatal error: concurrent map read and map write` 直接崩溃。

Go 中实现并发安全 map 有三种主流方案，各有优劣。我把三种都写了一遍，用同一套测试跑了一下，记录一下结果。

---

## 解法一：sync.RWMutex

### 思路

最传统、最稳妥的做法。用一个 `sync.RWMutex` 保护整个 map：

- **读操作**：加 `RLock()`（读锁之间不互斥）
- **写操作**：加 `Lock()`（独占）

### 实现

```go
type RWMutexMap struct {
    mu   sync.RWMutex
    data map[string]int
}

func NewRWMutexMap() *RWMutexMap {
    return &RWMutexMap{data: make(map[string]int)}
}

func (m *RWMutexMap) Get(key string) (int, bool) {
    m.mu.RLock()         // 读锁：允许多个 goroutine 同时持有
    defer m.mu.RUnlock()
    v, ok := m.data[key]
    return v, ok
}

func (m *RWMutexMap) Set(key string, val int) {
    m.mu.Lock()          // 写锁：独占
    defer m.mu.Unlock()
    m.data[key] = val
}

func (m *RWMutexMap) Delete(key string) {
    m.mu.Lock()
    defer m.mu.Unlock()
    delete(m.data, key)
}
```

### 优点

- 实现简单，代码量最小
- 逻辑直观，容易理解和维护
- `RWMutex` 在读多写少场景下性能可接受

### 缺点

- 只有一把全局锁，所有写操作串行化
- 高并发写入时，锁竞争成为瓶颈
- 读写锁本身有开销

---

## 解法二：sync.Map

### 思路

Go 标准库 `sync` 包内置的并发安全 map。接口是 `any` 类型的（无泛型），我们封装一层提供类型安全。

`sync.Map` 的底层采用了**读写分离**架构：

- **read map**（只读，无锁访问）：存储稳定的、不常变动的 key
- **dirty map**（需加锁访问）：存储新写入的 key

读取时先查 read map（lock-free），miss 后再查 dirty map。当 miss 次数达到阈值，dirty map 会晋升为新的 read map。

### 实现

```go
type SyncMapWrapper struct {
    m sync.Map
}

func (sm *SyncMapWrapper) Get(key string) (int, bool) {
    v, ok := sm.m.Load(key)
    if !ok {
        return 0, false
    }
    return v.(int), true
}

func (sm *SyncMapWrapper) Set(key string, val int) {
    sm.m.Store(key, val)
}

func (sm *SyncMapWrapper) Delete(key string) {
    sm.m.Delete(key)
}
```

### sync.Map 的适用场景

> The Map type is optimized for two common use cases:
>
> 1. **Write-once, read-many**：key 只写入一次，之后大量读取（如配置中心、缓存）
> 2. **Disjoint key sets**：多个 goroutine 操作的 key 集合互不相交（各读各的、各写各的）

### 优点

- 特定场景下读操作是 **lock-free** 的，性能极高
- 标准库自带，无需第三方依赖
- 针对上述两种场景高度优化

### 缺点

- 写操作需要加锁，且涉及 map 的复制和晋升，写密集时性能差
- 接口为 `any` 类型，需手动类型断言，无编译期类型安全
- 不适合 key 集合频繁变动的场景

---

## 解法三：分段锁 ShardMap

### 思路

当并发量极高（每秒几十万甚至百万级写请求）时，单把全局锁成为瓶颈。分段锁的核心思想：

1. 将一个大 map 拆分成 **32 个**（或更多）小的 shard
2. 每个 shard 有**独立的 RWMutex**
3. key 通过哈希函数路由到对应 shard
4. 操作时只锁目标 shard，不影响其他 shard

**效果：锁粒度降低 32 倍，竞争概率降低 32 倍。**

### 实现

```go
const numShards = 32

type shard struct {
    mu   sync.RWMutex
    data map[string]int
}

type ShardMap struct {
    shards [numShards]*shard
}

func NewShardMap() *ShardMap {
    sm := &ShardMap{}
    for i := 0; i < numShards; i++ {
        sm.shards[i] = &shard{data: make(map[string]int)}
    }
    return sm
}

// FNV-1a 哈希，快速且分布均匀
func fnv32(key string) uint32 {
    h := uint32(2166136261)
    for i := 0; i < len(key); i++ {
        h ^= uint32(key[i])
        h *= 16777619
    }
    return h
}

func (sm *ShardMap) getShard(key string) *shard {
    return sm.shards[fnv32(key)%numShards]
}

func (sm *ShardMap) Get(key string) (int, bool) {
    sh := sm.getShard(key)
    sh.mu.RLock()
    defer sh.mu.RUnlock()
    v, ok := sh.data[key]
    return v, ok
}

func (sm *ShardMap) Set(key string, val int) {
    sh := sm.getShard(key)
    sh.mu.Lock()
    defer sh.mu.Unlock()
    sh.data[key] = val
}
```

### 为什么选择 FNV-1a 哈希

- **速度快**：无内存分配，纯整数运算
- **分布均匀**：在 `%32` 取模场景下碰撞少
- 也可以使用标准库 `hash/fnv` 包，但会引入一次 `hash.Hash` 接口分配

### 为什么是 32 个 shard

- 是 2 的幂，取模可用位运算优化（编译器会对 `%32` 优化为 `&31`）
- 32 个锁的内存开销可以忽略
- 来自知名开源库 `orcaman/concurrent-map` 的实践

### 优点

- 高并发写入场景下性能最优
- 锁竞争大幅降低（理论降低 32 倍）
- 可扩展：调大 `numShards` 可进一步降低竞争

### 缺点

- 实现稍复杂
- `Len()` 等遍历操作需要锁住所有 shard
- 哈希计算有微小开销

---

## 测试策略与用例

### 测试方式

三种实现用同一套测试跑，定义了一个 `concurrentMap` 接口来避免重复代码：

```go
type concurrentMap interface {
    Get(key string) (int, bool)
    Set(key string, val int)
    Delete(key string)
}
```

### 测试用例清单

| 测试用例 | 场景 | 验证点 |
|---|---|---|
| `BasicSetGet` | 写入后读取 | 基本正确性 |
| `SetOverwrite` | 覆盖写入同一 key | 覆盖语义正确 |
| `Delete` | 删除存在的/不存在的 key | 删除后不可见，不存在不 panic |
| `GetNonExistent` | 读取不存在的 key | 返回 false |
| `ConcurrentReads` | 100 goroutine × 1000 次读同一 key | 并发读安全，值正确 |
| `ConcurrentWritesDiffKeys` | 100 goroutine 各写不同 key | 并发写不丢失数据 |
| `ConcurrentWritesSameKey` | 50 goroutine 抢写同一 key | 写不 panic，最终存在 |
| `MixedReadWrite` | 20 写 + 80 读，各 500 次 ops | 混合负载正确性 |
| `HighConcurrencyStress` | 200 goroutine × 1000 次混合 ops | 高压下不 panic |

### 额外测试

| 测试用例 | 说明 |
|---|---|
| `TestRWMutexMap_Len` | 验证 RWMutexMap 的 Len() 正确性 |
| `TestShardMap_Len` | 验证 ShardMap 的 Len() 跨 shard 统计 |
| `TestShardMap_Distribution` | 1000 个 key 在各 shard 的分布 |
| `TestXXX_Race` | 三个实现的专项竞态检测测试 |

---

## Benchmark 性能对比

### 测试方法

```go
func BenchmarkRWMutexMap_Set(b *testing.B) {
    m := NewRWMutexMap()
    b.RunParallel(func(pb *testing.PB) {
        i := 0
        for pb.Next() {
            m.Set(fmt.Sprintf("k%d", i), i)
            i++
        }
    })
}
```

每个 benchmark 使用 `b.RunParallel` 让所有 CPU 核心并发执行，模拟真实并发场景。

### 结果

**硬件：Apple M4 (10 核), Go 1.24**

| 操作 | RWMutexMap | sync.Map | ShardMap |
|---|---|---|---|
| **Set** | 166.1 ns/op | 59.2 ns/op | **62.4 ns/op** |
| **Get** | 92.8 ns/op | **1.4 ns/op** | 93.2 ns/op |

### 结果分析

**Set（写操作）**

RWMutexMap 最慢，166 ns，没什么意外——全局写锁，10 个核上的 goroutine 全部在排队等这一把锁。

SyncMap 和 ShardMap 分别是 59 ns 和 62 ns，都比 RWMutexMap 快了一截。SyncMap 写的时候需要操作 dirty map，在 miss 太多的时候还会触发 dirty map 晋升（把 dirty 复制一份变成新的 read map），不过在 key 各不相同的场景下晋升很少发生。ShardMap 就是把锁粒度拆小了，同一时刻不同 shard 可以各自处理写请求。

ShardMap 的写大概比 RWMutexMap 快 2.7 倍。

**Get（读操作）**

sync.Map 的读是 1.4 ns，这个数字很夸张。原因是它的 read map 存在 `atomic.Value` 里面，读的时候直接 load 出来然后查普通 map，完全不需要加锁。1.4 ns 在 M4 上大概就是 5-6 个 CPU 周期。

RWMutexMap 和 ShardMap 的读都是 93 ns。都需要走 `RLock()`，虽然读锁之间不互斥，但 `RLock` 本身是一次原子操作，有开销。93 ns 对于"加读锁→查 map→解读锁"来说挺正常的。

sync.Map 的读比另外两个快了大概 66 倍。这就是它读写分离架构的意义——把最频繁的读路径上的锁完全去掉了。

**为什么 sync.Map 的写也不慢？**

这次的 benchmark 里每个 key 都是 `k0`, `k1`, `k2`... 各不相同，正好符合 sync.Map 的第二个适用场景（disjoint key sets）。写入直接进 dirty map，晋升很少触发，所以写性能跟 ShardMap 差不多。但如果 key 集合不稳定、同一个 key 反复被不同的 goroutine 写，sync.Map 的写性能就会明显下降。

---

## 分片均匀性验证

运行 `TestShardMap_Distribution` 将 1000 个 key 写入 ShardMap，统计每个 shard 的数据量：

```
shard[0]:  35 keys    shard[11]: 33 keys    shard[22]: 30 keys
shard[1]:  27 keys    shard[12]: 35 keys    shard[23]: 30 keys
shard[2]:  30 keys    shard[13]: 27 keys    shard[24]: 33 keys
shard[3]:  30 keys    shard[14]: 27 keys    shard[25]: 33 keys
shard[4]:  30 keys    shard[15]: 31 keys    shard[26]: 27 keys
shard[5]:  38 keys    shard[16]: 30 keys    shard[27]: 26 keys
shard[6]:  33 keys    shard[17]: 36 keys    shard[28]: 31 keys
shard[7]:  29 keys    shard[18]: 38 keys    shard[29]: 30 keys
shard[8]:  26 keys    shard[19]: 35 keys    shard[30]: 36 keys
shard[9]:  30 keys    shard[20]: 29 keys    shard[31]: 35 keys
shard[10]: 30 keys    shard[21]: 30 keys

empty shards: 0/32
```

- **0 个空 shard**，32 个 shard 全用上了
- 每个 shard 26-38 个 key，分布比较均匀
- 理论均值 31.25（1000 ÷ 32），实际吻合

FNV-1a 这个哈希函数应付 `%32` 取模绰绰有余。

---

## 竞态检测

所有测试均通过 Go race detector：

```bash
$ go test -race -run "TestRWMutexMap$|TestSyncMapWrapper$|TestShardMap$" -v
=== RUN   TestRWMutexMap
--- PASS: TestRWMutexMap (0.26s)
=== RUN   TestSyncMapWrapper
--- PASS: TestSyncMapWrapper (0.11s)
=== RUN   TestShardMap
--- PASS: TestShardMap (0.11s)
PASS
```

`-race` 会在编译时插桩所有内存访问，运行时检测有没有并发的读写冲突。三个实现都干干净净，没有报任何 race。

---

## 如何选择

没有标准答案，看场景。

**如果你不确定自己是什么场景，直接用 RWMutexMap。** 代码最简单，出问题了最好排查。后面 profile 发现确实是瓶颈了，再换也不迟。

**如果你的服务是读极多写极少**（比如启动时加载配置，之后就是海量读取），sync.Map 的 lock-free 读很实在，1.4 ns 对比 93 ns 是实实在在的差距。

**如果你在写密集场景**（比如并发计数器、实时统计），而且 QPS 确实高到 RWMutexMap 扛不住了，ShardMap 的分段锁能让写吞吐好很多。

一些具体情况：

| 场景 | 推荐 | 原因 |
|---|---|---|
| 配置中心、缓存、词典 | sync.Map | key 稳定，读极多写极少 |
| 高并发计数器、实时统计 | ShardMap | 写密集，分段锁减少竞争 |
| 普通 Web 服务 | RWMutexMap | 代码简单，够用 |
| 需要 Len() 或 Range() | RWMutexMap / ShardMap | sync.Map 的 Range 要遍历 read+dirty，不好估算耗时 |
| 需要编译期类型安全 | RWMutexMap / ShardMap | sync.Map 用 `any`，值拿出来要断言 |
| 不确定 | RWMutexMap | 先写简单的，真遇到瓶颈再换 |

### 一些补充

1. ShardMap 的 shard 数量不一定要 32。一般 32-256 都合理。太多了浪费内存，太少了竞争还是严重。
2. sync.Map 不适合频繁增删 key。它删 key 只是打标记，不会立刻清理。dirty map 晋升的时候，被标记删除的 key 不会被复制过去，但这个过程不可控。
3. Go 1.22 之后 sync.Map 加了 `CompareAndSwap`、`Swap` 等方法，比以前好用一些。
4. 本文用 `map[string]int` 是为了简单，实际项目建议写泛型版本 `map[K comparable, V any]`。

---

## 完整源码

### map.go — 三种实现

```go
package main

import "sync"

// ========== 解法一：sync.RWMutex ==========

type RWMutexMap struct {
    mu   sync.RWMutex
    data map[string]int
}

func NewRWMutexMap() *RWMutexMap {
    return &RWMutexMap{data: make(map[string]int)}
}

func (m *RWMutexMap) Get(key string) (int, bool) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    v, ok := m.data[key]
    return v, ok
}

func (m *RWMutexMap) Set(key string, val int) {
    m.mu.Lock()
    defer m.mu.Unlock()
    m.data[key] = val
}

func (m *RWMutexMap) Delete(key string) {
    m.mu.Lock()
    defer m.mu.Unlock()
    delete(m.data, key)
}

func (m *RWMutexMap) Len() int {
    m.mu.RLock()
    defer m.mu.RUnlock()
    return len(m.data)
}

// ========== 解法二：sync.Map 封装 ==========

type SyncMapWrapper struct {
    m sync.Map
}

func NewSyncMapWrapper() *SyncMapWrapper {
    return &SyncMapWrapper{}
}

func (sm *SyncMapWrapper) Get(key string) (int, bool) {
    v, ok := sm.m.Load(key)
    if !ok {
        return 0, false
    }
    return v.(int), true
}

func (sm *SyncMapWrapper) Set(key string, val int) {
    sm.m.Store(key, val)
}

func (sm *SyncMapWrapper) Delete(key string) {
    sm.m.Delete(key)
}

// ========== 解法三：分段锁 ShardMap ==========

const numShards = 32

type shard struct {
    mu   sync.RWMutex
    data map[string]int
}

type ShardMap struct {
    shards [numShards]*shard
}

func NewShardMap() *ShardMap {
    sm := &ShardMap{}
    for i := 0; i < numShards; i++ {
        sm.shards[i] = &shard{data: make(map[string]int)}
    }
    return sm
}

func fnv32(key string) uint32 {
    h := uint32(2166136261)
    for i := 0; i < len(key); i++ {
        h ^= uint32(key[i])
        h *= 16777619
    }
    return h
}

func (sm *ShardMap) getShard(key string) *shard {
    return sm.shards[fnv32(key)%numShards]
}

func (sm *ShardMap) Get(key string) (int, bool) {
    sh := sm.getShard(key)
    sh.mu.RLock()
    defer sh.mu.RUnlock()
    v, ok := sh.data[key]
    return v, ok
}

func (sm *ShardMap) Set(key string, val int) {
    sh := sm.getShard(key)
    sh.mu.Lock()
    defer sh.mu.Unlock()
    sh.data[key] = val
}

func (sm *ShardMap) Delete(key string) {
    sh := sm.getShard(key)
    sh.mu.Lock()
    defer sh.mu.Unlock()
    delete(sh.data, key)
}

func (sm *ShardMap) Len() int {
    total := 0
    for i := 0; i < numShards; i++ {
        sm.shards[i].mu.RLock()
        total += len(sm.shards[i].data)
        sm.shards[i].mu.RUnlock()
    }
    return total
}
```

### 运行测试

```bash
# 运行所有并发 map 测试
go test -v -run "TestRWMutexMap|TestSyncMapWrapper|TestShardMap" -race

# 运行 benchmark
go test -bench="Benchmark(RWMutexMap|SyncMapWrapper|ShardMap)" -benchtime=1s -run="^$"
```

---

> 一句话总结：要简单用 RWMutexMap，读多用 sync.Map，写多用 ShardMap。拿不准就先写 RWMutexMap，profile 发现瓶颈再说。
