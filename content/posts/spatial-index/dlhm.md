---
date: '2026-03-17T10:00:00+08:00'
draft: false
title: '学习型空间索引设计与实践'
tags: ["Database", "Research", "ML", "Learned Index"]
description: "密度感知的 Hilbert 映射降维方法、误差受控的空间查询、面向动态更新的索引维护"
toc: false
katex: true
---

> 空间索引是数据库系统的核心组件。传统方法（R-tree, Hilbert 曲线）在高偏态分布和动态更新场景下面临秩值簇化和长尾延迟的挑战。本文提出三阶段的学习型空间索引方案。

---

## 1. DLHM：密度感知的 Hilbert 降维映射

### 1.1 问题定义

给定 $d$ 维空间数据集 $\mathcal{D} = \{\mathbf{p}_1, \dots, \mathbf{p}_N\}$，其中 $\mathbf{p}_i \in \mathbb{R}^d$（地理空间场景下通常 $d = 2$）。目标是将 $\mathcal{D}$ 映射到一维有序序列 $\pi: \mathcal{D} \to [0, M-1]$，使得空间邻近性在一维序列中得以保持：

$$
\mathbb{E}_{(\mathbf{p}_i, \mathbf{p}_j) \sim \mathcal{D}}\left[|\pi(\mathbf{p}_i) - \pi(\mathbf{p}_j)| \;\middle|\; \|\mathbf{p}_i - \mathbf{p}_j\|_2 \leq \epsilon\right] \to \min
$$

### 1.2 密度分布形式化模型

定义空间密度函数 $\rho(\mathbf{x})$ 为位置 $\mathbf{x}$ 处的局部数据密度。采用核密度估计（KDE）：

$$
\rho(\mathbf{x}) = \frac{1}{N h^d} \sum_{i=1}^{N} K\left(\frac{\|\mathbf{x} - \mathbf{p}_i\|}{h}\right)
$$

其中 $K(\cdot)$ 为 Epanechnikov 核函数，$h$ 为带宽参数，按 Silverman 规则 $h = 1.06 \cdot \hat{\sigma} \cdot N^{-1/5}$ 确定。

在每个空间区域 $R_k$ 内，定义局部密度 $D_k^{\text{local}}$ 与全局平均密度 $\bar{D}$ 的比值：

$$
\lambda_k = \frac{D_k^{\text{local}}}{\bar{D}},\quad D_k^{\text{local}} = \int_{R_k} \rho(\mathbf{x})\,d\mathbf{x},\quad \bar{D} = \frac{1}{K}\sum_{j=1}^{K}D_j^{\text{local}}
$$

### 1.3 自适应四叉树分裂策略

DLHM 采用自顶向下的四叉树分割。分裂判据基于密度比值 $\lambda_k$：

$$
\text{Split}(R_k) =
\begin{cases}
\text{true},  & \lambda_k > \lambda_{\text{high}} \land \text{depth}(R_k) < d_{\max} \\
\text{false}, & \lambda_k < \lambda_{\text{low}} \lor \text{depth}(R_k) \geq d_{\max} \\
\text{continue}, & \text{otherwise}
\end{cases}
$$

其中 $\lambda_{\text{high}} = 2.0$（过密触发分裂），$\lambda_{\text{low}} = 0.3$（过疏停止），$d_{\max} = 12$（最大深度限制）。

分裂后的叶节点集合 $\mathcal{L} = \{L_1, \dots, L_m\}$ 满足空间互斥且完备覆盖。

### 1.4 局部 Hilbert 编码

在叶节点 $L_k$ 内部，独立执行 Hilbert 编码。对点 $\mathbf{p} = (x, y)$，首先归一化到 $[0, 2^r - 1]^2$ 的整数网格：

$$
(x_g, y_g) = \left(\left\lfloor \frac{x - x_k^{\min}}{x_k^{\max} - x_k^{\min}} \cdot (2^r - 1) \right\rfloor, \left\lfloor \frac{y - y_k^{\min}}{y_k^{\max} - y_k^{\min}} \cdot (2^r - 1) \right\rfloor\right)
$$

Hilbert 编码 $H(x_g, y_g)$ 通过位交错和 Gray 码转换计算。定义阶数为 $r$，编码空间大小为 $2^{2r}$。

### 1.5 跨网格偏移注入机制

定义全局偏移量 $\Delta_k$ 为前 $k-1$ 个叶节点的编码容量之和：

$$
\Delta_k = \sum_{j=1}^{k-1} \left\lceil \lambda_j \cdot \frac{2^{2r}}{m} \right\rceil
$$

每个叶节点按密度比值 $\lambda_j$ 分配编码容量。最终全局映射：

$$
\pi(\mathbf{p}) = \Delta_k + H(x_g, y_g),\quad \mathbf{p} \in L_k
$$

保证了 $\pi$ 的全局单调递增性：若 $a < b$（按四叉树深度优先遍历顺序），则 $\forall \mathbf{p}_a \in L_a, \forall \mathbf{p}_b \in L_b$，有 $\pi(\mathbf{p}_a) < \pi(\mathbf{p}_b)$。

### 1.6 映射质量度量

**间隙方差（Gap Variance）**：衡量一维序列中点间距离的不均匀程度。

记 $\delta_i = \pi_{(i+1)} - \pi_{(i)}$ 为相邻秩值差（其中 $\pi_{(i)}$ 为有序排列），则：

$$
\text{GapVar}(\pi) = \frac{1}{N-1}\sum_{i=1}^{N-1}\left(\delta_i - \bar{\delta}\right)^2,\quad \bar{\delta} = \frac{1}{N-1}\sum_{i=1}^{N-1}\delta_i
$$

**映射平滑度（Smoothness）**：衡量空间邻近点在一维序列中保持邻近的程度。

记 $\mathcal{N}_{\epsilon}(\mathbf{p}_i)$ 为 $\mathbf{p}_i$ 的 $\epsilon$-邻域，定义：

$$
\text{Smooth}(\pi) = \frac{1}{N}\sum_{i=1}^{N}\frac{|\pi(\mathcal{N}_{\epsilon}(\mathbf{p}_i))|_{\text{span}}}{|\pi(\mathcal{N}_{\epsilon}(\mathbf{p}_i))|}
$$

其中 $|\cdot|_{\text{span}}$ 为最大秩差，$|\cdot|$ 为邻域内点数。

### 1.7 实验结论

| 指标 | 全局 Hilbert | DLHM | 提升 |
|---|---|---|---|
| Gap Variance | 1.00× | 0.26× | **-74.0%** |
| Smoothness | 1.0× | 2.3× | **+130%** |
| 秩值簇化指数 | 0.73 | 0.11 | **-85%** |

---

## 2. ETSQ：误差受控的两阶段空间查询

### 2.1 查询代价形式化

对于空间范围查询 $Q(\mathbf{q}, r) = \{\mathbf{p} \in \mathcal{D} \mid \|\mathbf{p} - \mathbf{q}\|_2 \leq r\}$，查询代价由推理开销和扫描开销组成：

$$
\mathcal{C}(Q) = \underbrace{t_{\text{inf}} \cdot C_{\text{inf}}}_{\text{推理开销}} + \underbrace{t_{\text{scan}} \cdot C_{\text{scan}}(r)}_{\text{扫描开销}}
$$

其中：
- $C_{\text{inf}}$：索引推理的常数开销（MLP 前向传播时间）
- $C_{\text{scan}}(r) = \alpha \cdot \pi r^2 \cdot \beta$：扫描开销，与查询面积成正比
- $t_{\text{inf}}$ 和 $t_{\text{scan}}$ 为对应操作次数

**核心权衡**：增大候选区间 → 减少推理次数但增加扫描量；缩小候选区间 → 反之。

### 2.2 异构两阶段架构

#### Phase 1: 轻量级 MLP 路由层

路由函数 $f_{\theta}: \mathbb{R}^d \to \mathbb{R}$ 以亚线性开销将查询点映射到粗粒度区间：

$$
\hat{I} = [f_{\theta}(\mathbf{q}) - \Delta_L,\; f_{\theta}(\mathbf{q}) + \Delta_R]
$$

其中 MLP 结构为 $d \to 64 \to 32 \to 1$（GELU 激活），参数量精确控制在 $\Theta = 64d + 64 + 32 \cdot 64 + 32 + 32 + 1$。

训练目标为最小化区间覆盖误差：

$$
\mathcal{L}_{\text{route}} = \frac{1}{|\mathcal{D}|}\sum_{\mathbf{p} \in \mathcal{D}}\left[\text{ReLU}(\pi(\mathbf{p}) - f_{\theta}(\mathbf{p}) - \Delta) + \text{ReLU}(f_{\theta}(\mathbf{p}) - \Delta - \pi(\mathbf{p}))\right]
$$

#### Phase 2: $\varepsilon$-有界分段线性回归定位层

在候选区间 $\hat{I}$ 内，使用分段线性函数 $g_{\phi}$ 精确定位：

$$
\pi(\mathbf{q}) \approx g_{\phi}(\mathbf{q}) = \sum_{s=1}^{S} w_s \cdot \text{ReLU}\left(\mathbf{v}_s^{\top}\mathbf{q} + b_s\right)
$$

关键约束：预测误差有界。定义 $\varepsilon$ 边界保证：

$$
\max_{\mathbf{p} \in \mathcal{D}}|g_{\phi}(\mathbf{p}) - \pi(\mathbf{p})| \leq \varepsilon
$$

训练时通过双目标优化强制满足：

$$
\mathcal{L}_{\text{loc}} = \frac{1}{N}\sum_{i=1}^{N}(\pi(\mathbf{p}_i) - g_{\phi}(\mathbf{p}_i))^2 + \lambda \cdot \max(0, |\pi(\mathbf{p}_i) - g_{\phi}(\mathbf{p}_i)| - \varepsilon)^2
$$

其中 $\lambda = 10.0$ 为约束违反惩罚系数，$\varepsilon = 16$（容许误差 16 个秩值位）。

### 2.3 查询执行流程

```
输入：查询点 q，半径 r
阶段 1 — MLP 路由：
    pid_hat = f_θ(q);
    候选区间 = [pid_hat - Δ_L - ε, pid_hat + Δ_R + ε]
    候选点集 = 区间内所有点
阶段 2 — 精确过滤：
    对候选集中每个点 p：
        if ||p - q|| ≤ r → 加入结果集
返回结果
```

### 2.4 实验结论

**硬件：Intel Xeon 8358P × 2（64 核），数据集规模 $10^7$ 条**

| 方法 | P50 延迟 | P99 延迟 | 吞吐量 (QPS) |
|---|---|---|---|
| R-tree | 4.20 μs | 18.7 μs | 2.38×10⁴ |
| B+tree (Hilbert) | 3.15 μs | 15.2 μs | 3.17×10⁴ |
| RSMI | 2.80 μs | 9.43 μs | 3.57×10⁴ |
| LISA | 2.10 μs | 7.81 μs | 4.76×10⁴ |
| **ETSQ** | **1.35 μs** | **4.80 μs** | **7.41×10⁴** |

在各选择度下（$10^{-6}$ ~ $10^{-1}$），ETSQ 的延迟保持稳定，长尾受控。

---

## 3. DLSM：动态更新下的索引维护

### 3.1 预测偏移代价模型

动态更新场景下，插入/删除操作导致数据分布漂移，模型预测误差逐渐增大。定义时刻 $t$ 的结构化漂移指标：

$$
\Psi_t = \frac{1}{|\mathcal{D}_t|}\sum_{\mathbf{p} \in \mathcal{D}_t} \left|\pi_t(\mathbf{p}) - \hat{\pi}_t(\mathbf{p})\right|
$$

其中 $\pi_t$ 为理想秩值，$\hat{\pi}_t$ 为当前模型预测值。

局部重构触发条件：

$$
\Psi_t^{(k)} = \frac{1}{|L_k|}\sum_{\mathbf{p} \in L_k}\left|\pi_t(\mathbf{p}) - \hat{\pi}_t(\mathbf{p})\right| > \tau_{\text{reorg}},\quad \tau_{\text{reorg}} = 32
$$

### 3.2 Gapped Array 存储结构

每个叶节点的数据以 Gapped Array 组织。定义填充因子 $\phi_k$（已占用比例）和空闲率 $\gamma_k = 1 - \phi_k$：

$$
\phi_k = \frac{|L_k|}{\text{capacity}(L_k)}
$$

自适应空闲率分配策略根据节点密度比值 $\lambda_k$ 动态调整：

$$
\gamma_k^{\text{target}} = \gamma_{\min} + (\gamma_{\max} - \gamma_{\min}) \cdot \frac{\log \lambda_k}{\log \lambda_{\max}}
$$

其中 $\gamma_{\min} = 0.05$，$\gamma_{\max} = 0.50$，$\lambda_{\max}$ 为所有叶节点最大密度比值。

该策略保证高密度区域留有充足空位吸收插入，低密度区域避免空间浪费。

### 3.3 写时复制快照隔离（CoW-SI）

DLSM 采用写时复制快照隔离机制：读操作访问当前快照 $S_{\text{cur}}$，写操作在影子副本 $S_{\text{shadow}}$ 上进行。

快照状态机：

$$
S_{\text{cur}} \xrightarrow[\text{shadow merge}]{\text{async}} S_{\text{next}}
$$

异步重构与原子替换流程：

```
1. 创建当前四叉树 T_cur 的影子副本 T_shadow
2. 在 T_shadow 上批量执行写操作（插入/删除/更新）
3. 检查 Ψ_t 是否超过 τ_reorg
   → 若超过：对受影响的叶节点集 L_dirty 执行局部重构
   → 重新训练 MLP 路由层（增量 SGD，5 epochs）
4. 更新 T_shadow 中的 Gapped Array 布局
5. 原子 CAS 切换：T_cur ← T_shadow（通常在微秒级完成）
```

### 3.4 增量训练策略

MLP 路由层的更新采用增量 SGD，仅对受影响样本训练：

$$
\theta_{t+1} = \theta_t - \eta \cdot \frac{1}{|\mathcal{D}_{\text{delta}}|}\sum_{\mathbf{p} \in \mathcal{D}_{\text{delta}}} \nabla_{\theta} \mathcal{L}_{\text{route}}(\mathbf{p}; \theta_t)
$$

其中 $\mathcal{D}_{\text{delta}}$ 为增量数据（新增 + 删除影响的邻居），$\eta = 0.001$。

### 3.5 实验结论

**硬件：Intel Xeon 8358P × 2（64 核），OS：Ubuntu 22.04**

| 负载类型 | 吞吐量 (Ops/sec) | P50 读 | P50 写 |
|---|---|---|---|
| R100/W0 | 2.86×10⁵ | 2.1 μs | — |
| R50/W50 | **1.18×10⁵** | 3.8 μs | 4.7 μs |
| R0/W100 | 1.43×10⁵ | — | 3.5 μs |

72 小时持续运行稳定性：$\Psi_t < \tau_{\text{reorg}}$ 的时间占比 99.97%，最大漂移 $\Psi_{\max} = 28.7 < 32$（重构阈值）。

### 3.6 与对比方法的读写平衡性

定义读写平衡性指标 $B$：

$$
B = \frac{\min(\text{Throughput}_{\text{read}}, \text{Throughput}_{\text{write}})}{\max(\text{Throughput}_{\text{read}}, \text{Throughput}_{\text{write}})}
$$

| 方法 | R50/W50 吞吐量 | 平衡性 $B$ |
|---|---|---|
| R-tree (R\*L) | 4.32×10⁴ | 0.52 |
| B+tree (Hilbert) | 5.17×10⁴ | 0.48 |
| RSMI | 5.83×10⁴ | 0.61 |
| LISA | 6.21×10⁴ | 0.55 |
| **DLSM** | **1.18×10⁵** | **0.78** |

---

## 4. 总结

DLHM、ETSQ 与 DLSM 构成了学习型空间索引的完整方案：

- **DLHM** 解决了高偏态分布下的秩值簇化，间隙方差降低 74%
- **ETSQ** 实现了亚微秒级查询延迟（P50 1.35 μs），长尾受控
- **DLSM** 在动态更新场景下实现 10⁵ Ops/sec 吞吐，读写平衡性 0.78

核心设计原则：密度感知的自适应分区、异构两阶段的误差受控查询、写时复制快照隔离的异步维护。三者协同，从静态降维、高效查询到动态维护，完整覆盖了空间索引的全生命周期。
