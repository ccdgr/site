---
date: '2024-02-14T11:20:00+08:00'
draft: false
title: 'Go Slice 底层结构与使用陷阱'
tags: ["Go"]
description: "Go slice 底层结构、扩容机制、截取与底层数组共享、append 与 copy 的正确用法"
toc: false
---

> Slice 是 Go 最常用的数据结构，但也是坑最多的地方。

---

## 底层结构

```go
type slice struct {
    array unsafe.Pointer // 指向底层数组的指针
    len   int            // 当前长度
    cap   int            // 容量
}
```

```
slice := make([]int, 3, 5)

┌────────┐
│ array  │─────→  ┌───┬───┬───┬───┬───┐
│ len=3  │       │ 0 │ 0 │ 0 │ ? │ ? │
│ cap=5  │       └───┴───┴───┴───┴───┘
└────────┘        ←── len ──→
                  ←────── cap ──────→
```

## 扩容机制

当 `append` 时 len == cap，runtime 会分配新数组：

```go
s := []int{1, 2}      // len=2, cap=2
s = append(s, 3)       // 触发扩容
// cap 变为 2 → 4 → 8 ...（翻倍增长，大切片 1.25 倍）
```

| Go 版本 | 扩容策略 |
|---|---|
| < 1.17 | 1024 以下翻倍，以上增长 1.25 倍 |
| ≥ 1.18 | 改为更平滑的策略，按预期增长量对齐内存 |

```go
// 查看扩容行为
s := make([]int, 0)
for i := 0; i < 10; i++ {
    s = append(s, i)
    fmt.Printf("len=%d cap=%d\n", len(s), cap(s))
}
// len=1 cap=1
// len=2 cap=2
// len=3 cap=4
// len=4 cap=4
// len=5 cap=8
// ...
```

扩容意味着**底层数组迁移**，原指针会失效。

## 截取与底层共享

这是 slice 最大的坑：

```go
s1 := []int{1, 2, 3, 4, 5}
s2 := s1[1:3]  // s2 = [2, 3], len=2, cap=4（共享底层数组！）

s2[0] = 99
fmt.Println(s1)  // [1, 99, 3, 4, 5] ← s1 也被改了！

s2 = append(s2, 100)
fmt.Println(s1)  // [1, 99, 100, 4, 5] ← s1 又被改了！
```

```
底层数组：  ┌───┬───┬───┬───┬───┐
            │ 1 │ 2 │ 3 │ 4 │ 5 │
            └───┴───┴───┴───┴───┘
             ↑       ↑
s1:         从 0 开始，len=5, cap=5
s2:              从 1 开始，len=2, cap=4
```

### 避坑：用 copy 或 full slice

```go
// 方式一：完整截取 + append
s2 := append([]int{}, s1[1:3]...)

// 方式二：copy
s2 := make([]int, 2)
copy(s2, s1[1:3])

// 方式三：限制 cap
s2 := s1[1:3:3]  // len=2, cap=2（第三个值限制 cap）
```

## append 的常见误解

```go
func add(s []int) {
    s = append(s, 99)  // 只改函数内的 s，外部不受影响
}

func addPtr(s *[]int) {
    *s = append(*s, 99)  // 传指针才影响外部
}

func addAndReturn(s []int) []int {
    return append(s, 99)  // 返回新 slice
}
```

`append` 返回的是**新 slice header**（底层数组可能变了），所以必须接收返回值。

## nil slice vs empty slice

```go
var s1 []int           // nil slice，len=0, cap=0, array=nil
s2 := []int{}          // empty slice，len=0, cap=0, array 有地址
s3 := make([]int, 0)   // empty slice

fmt.Println(s1 == nil) // true
fmt.Println(s2 == nil) // false

// JSON 编码差异
json.Marshal(s1)  // null
json.Marshal(s2)  // []
```

大多数情况下 `len(s) == 0` 判断就够了，不需要区分 nil 和 empty。但 **JSON 序列化、反射、protobuf** 场景需要注意。

## 预分配优化

```go
// 差：多次扩容 + 内存迁移
var s []int
for i := 0; i < 10000; i++ {
    s = append(s, i)
}

// 好：一次分配
s := make([]int, 0, 10000)
for i := 0; i < 10000; i++ {
    s = append(s, i)
}
```

## 总结

- slice 是 header（array + len + cap）+ 底层数组
- 截取共享底层数组，改一个会影响另一个
- `append` 必须接收返回值
- 预分配 capacity 减少扩容
- `s[1:3:3]` 第三个值限制 cap，防止共享
- 对外暴露 slice 时，传 copy 而非原数据
