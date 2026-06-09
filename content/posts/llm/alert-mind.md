---
date: '2026-03-11T00:00:00+08:00'
draft: false
title: 'AlertMind 智能异常诊断平台'
tags: ["LLM", "Agent"]
description: "AlertMind是一款专为云原生环境设计的自动化告警根因分析系统"
---

## 目录

1. [系统需求分析](#1-系统需求分析)
2. [核心业务流程](#2-核心业务流程)
3. [系统架构](#3-系统架构)
4. [模块划分](#4-模块划分)
5. [数据流设计](#5-数据流设计)
6. [数据库表设计](#6-数据库表设计)
7. [Redis 使用场景](#7-redis-使用场景)
8. [RabbitMQ 使用场景](#8-rabbitmq-使用场景)
9. [Agent 设计](#9-agent-设计)
10. [Tool Calling 设计](#10-tool-calling-设计)
11. [RAG 设计](#11-rag-设计)
12. [MVP 最小可行版本范围](#12-mvp-最小可行版本范围)
13. [技术选型总览](#13-技术选型总览)
14. [项目目录结构](#14-项目目录结构)

---

## 1. 系统需求分析

### 1.1 功能性需求

| 编号 | 需求 | 优先级 | 说明 |
|------|------|--------|------|
| F1 | 告警接入 | P0 | 接收 Prometheus AlertManager / Grafana / 自定义 Webhook 告警 |
| F2 | 日志分析 | P0 | 自动查询 Loki / Elasticsearch，提取异常上下文日志 |
| F3 | 指标分析 | P0 | 自动查询 Prometheus / VictoriaMetrics，获取异常时段指标 |
| F4 | 调用链分析 | P1 | 自动查询 Jaeger / Tempo，定位异常 Span |
| F5 | ReAct Agent 根因分析 | P0 | 基于 Thought-Action-Observation 循环的智能诊断 |
| F6 | 结构化诊断报告 | P0 | 输出包含根因、影响范围、修复建议的报告 |
| F7 | 历史案例检索 (RAG) | P1 | 基于语义相似度的历史案例匹配 |
| F8 | Web 管理界面 | P1 | 告警列表、报告查看、数据源配置 |

### 1.2 非功能性需求

| 编号 | 需求 | 说明 |
|------|------|------|
| NF1 | 单体部署 | 单个 Go 二进制文件，减少运维负担 |
| NF2 | 最小外部依赖 | 仅 PostgreSQL 为必需依赖 |
| NF3 | LLM 可替换 | 支持 OpenAI / Ollama / 任何 OpenAI 兼容 API |
| NF4 | 资源低消耗 | 空闲内存 < 100MB，适合中小团队资源 |
| NF5 | 配置驱动 | 数据源、告警规则均通过 YAML 或 UI 配置 |

### 1.3 不做的事（非目标）

- **不替代** Prometheus/Grafana/Loki/Jaeger — 只读查询，不做数据采集
- **不替代** AlertManager — 只做告警消费者
- **不提供**实时监控大盘 — 这不是监控系统
- **不支持**多租户 — 单团队使用

---

## 2. 核心业务流程

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  告警源       │    │  AlertMind   │    │  数据源       │
│ AlertManager │───▶│  告警接收      │    │              │
│ Webhook      │    │      │        │    │ Prometheus   │
└──────────────┘    │      ▼        │    │ Loki         │
                    │  数据收集      │───▶│ Jaeger       │
                    │      │        │    │ Kubernetes   │
                    │      ▼        │    └──────────────┘
                    │  Agent分析    │
                    │      │        │    ┌──────────────┐
                    │      ▼        │    │  LLM         │
                    │  报告生成      │◀───│ OpenAI/Ollama│
                    │      │        │    └──────────────┘
                    │      ▼        │
                    │  结果存储      │
                    │  通知用户      │
                    └──────────────┘
```

### 2.1 主流程（以一条告警为例）

```
1. 告警接入
   AlertManager 发送 webhook → AlertMind 接收 → 解析告警 → 写入 DB

2. 触发分析（同步或异步）
   创建分析任务 → 入队 → Worker 取出执行

3. 数据收集（Agent 驱动）
   Agent 根据告警信息决定需要哪些数据：
   - "先查一下这个服务的最近日志" → 调用 query_logs tool
   - "CPU 飙高，查一下同时段的 QPS" → 调用 query_metrics tool
   - "看看有没有慢调用" → 调用 query_traces tool

4. ReAct 循环分析
   Thought → Action → Observation → Thought → ... → Final Answer

5. 报告生成
   结构化输出：根因、证据链、影响范围、修复建议

6. 后处理
   存入 historical_cases 表 → 生成 embedding → 通知用户
```

### 2.2 RAG 增强流程

```
分析过程中 Agent 调用 search_cases tool:
  "有没有类似的告警案例？"
  → 当前告警描述 → embedding → pgvector 相似度搜索
  → 返回 Top-K 历史案例 → 注入 Agent 上下文
  → Agent 参考历史案例的处理方式加速诊断
```

---

## 3. 系统架构

### 3.1 架构图 (Mermaid)

```mermaid
graph TB
    subgraph External["外部系统"]
        AM[AlertManager]
        WH[Webhook Sources]
        PROM[Prometheus]
        LOKI[Loki / ES]
        JAEGER[Jaeger / Tempo]
        LLM[LLM API<br/>OpenAI / Ollama]
        NOTIFY[通知渠道<br/>飞书/钉钉/邮件]
    end

    subgraph AlertMind["AlertMind 单体应用"]
        subgraph HTTP["HTTP Layer (Gin)"]
            API[API Router]
            WEB[Web UI Static]
        end

        subgraph Core["Core Services"]
            AR[Alert Receiver<br/>告警接收与解析]
            DC[Data Collector<br/>数据适配器层]
            AE[Agent Engine<br/>ReAct Agent 执行引擎]
            RG[Report Generator<br/>报告生成器]
            RAG[RAG Engine<br/>案例检索]
        end

        subgraph Background["Background Workers"]
            WK[Analysis Worker<br/>异步分析执行]
            EMB[Embedding Worker<br/>案例向量化]
        end

        subgraph Store["Storage"]
            PG[(PostgreSQL<br/>+ pgvector)]
            RD[(Redis<br/>可选)]
        end
    end

    AM --> AR
    WH --> AR
    AR --> PG
    AR --> WK
    WK --> AE
    AE --> DC
    AE --> RAG
    AE --> LLM
    DC --> PROM
    DC --> LOKI
    DC --> JAEGER
    RAG --> PG
    AE --> RG
    RG --> PG
    RG --> EMB
    EMB --> PG
    RG --> NOTIFY

    API --> PG
    API --> RD
    WEB --> API
```

### 3.2 架构说明

**单体应用**，所有组件编译为单个 Go 二进制文件。启动后同时运行：
- HTTP Server（API + 静态资源）
- Background Worker Pool（内存级 goroutine pool，非独立进程）

**为什么不用微服务？**
- 个人开发者维护多个服务成本高
- 中小团队流量不需要独立扩缩容
- Go 的 goroutine 能很好处理并发
- 后续拆分成本低（按模块拆即可）

**依赖层级：**
```
必需: PostgreSQL（含 pgvector 扩展）
可选: Redis（缓存、限流）
可选: RabbitMQ（后续横向扩展时引入）
```

---

## 4. 模块划分

```
AlertMind
├── cmd/server/          # 应用入口
├── internal/
│   ├── alert/           # 告警接收与解析模块
│   │   ├── receiver.go      # Webhook handler
│   │   ├── parser.go        # 多源告警格式解析
│   │   └── normalizer.go    # 告警标准化
│   │
│   ├── collector/       # 数据收集适配器层
│   │   ├── interface.go     # DataCollector 接口
│   │   ├── loki.go          # Loki 日志查询
│   │   ├── prometheus.go    # Prometheus 指标查询
│   │   └── jaeger.go        # Jaeger 链路查询
│   │
│   ├── agent/           # ReAct Agent 引擎
│   │   ├── engine.go        # ReAct 循环执行器
│   │   ├── prompt.go        # System Prompt 管理
│   │   ├── tool_registry.go # 工具注册表
│   │   └── memory.go        # Agent 上下文窗口管理
│   │
│   ├── tools/           # Tool 实现
│   │   ├── query_logs.go
│   │   ├── query_metrics.go
│   │   ├── query_traces.go
│   │   ├── search_cases.go
│   │   └── get_topology.go
│   │
│   ├── rag/             # RAG 引擎
│   │   ├── embedder.go      # Embedding 生成
│   │   ├── retriever.go     # 向量检索
│   │   └── store.go         # 向量存储 (pgvector)
│   │
│   ├── report/          # 报告模块
│   │   ├── generator.go     # 结构化报告生成
│   │   ├── template.go      # 报告模板
│   │   └── notifier.go      # 通知发送
│   │
│   ├── llm/             # LLM 客户端
│   │   ├── client.go        # OpenAI 兼容客户端
│   │   └── types.go         # 消息/响应类型
│   │
│   └── store/           # 数据访问层
│       ├── postgres.go
│       └── migrations/
│
├── config/              # 配置文件
│   └── config.yaml
├── web/                 # 前端静态资源 (React/Vue)
└── DESIGN.md            # 本文档
```

**模块职责边界：**

| 模块 | 职责 | 依赖 |
|------|------|------|
| alert | 接收、解析、标准化告警 | store |
| collector | 统一数据源查询接口 | 无（纯外部调用） |
| agent | ReAct 循环控制、工具调度 | tools, llm |
| tools | 具体工具实现，暴露给 Agent | collector, rag |
| rag | 案例 embedding + 检索 | store, llm |
| report | 报告模板填充、通知 | store |
| llm | LLM API 封装 | 无 |
| store | 数据库 CRUD | PostgreSQL |

---

## 5. 数据流设计

### 5.1 告警分析主数据流

```
       ┌─────────┐
       │ 告警源   │
       └────┬────┘
            │ POST /api/v1/alerts
            ▼
    ┌───────────────┐
    │ Alert Receiver │ ──► 解析告警格式 → 标准化为 Alert 结构
    └───────┬───────┘
            │ INSERT INTO alerts
            ▼
    ┌───────────────┐
    │  PostgreSQL    │ ◄── alerts 表，status = pending
    └───────┬───────┘
            │ 创建分析任务
            ▼
    ┌───────────────┐
    │ 分析任务入队    │ ──► Go channel (MVP) 或 RabbitMQ (扩展)
    └───────┬───────┘
            │ Worker 取出
            ▼
    ┌───────────────┐
    │ Agent Engine   │
    │                │
    │ 循环:          │
    │  Thought: "需要查日志"         ──► LLM API
    │  Action: query_logs(...)       ──► Tools
    │  Observation: "发现大量 5xx"    ◄── Tools 返回
    │  ...                           ──► LLM API
    │  Final Answer: "{根因分析}"     ◄── LLM API
    └───────┬───────┘
            │ 结构化输出
            ▼
    ┌───────────────┐
    │ Report Gen     │ ──► 写入 analysis_reports
    └───────┬───────┘
            │ INSERT INTO historical_cases
            ▼
    ┌───────────────┐
    │ Embedding Gen  │ ──► 调用 embedding API → 写入 case_embeddings
    └───────┬───────┘
            │
            ▼
    ┌───────────────┐
    │ Notifier       │ ──► 飞书/钉钉/邮件通知
    └───────────────┘
```

### 5.2 RAG 检索数据流

```
Agent 调用 search_cases:
    当前告警描述
        │
        ▼
    embedder.Embed(description)  ──► 调用 LLM Embedding API
        │
        ▼
    pgvector 相似度搜索:
        SELECT * FROM case_embeddings
        ORDER BY embedding <=> query_vector
        LIMIT 5
        │
        ▼
    返回 Top-K 历史案例 → 注入 Agent 上下文
```

### 5.3 Agent 工具调用时序

```
Agent                     LLM                    Tools
  │                        │                       │
  │── {告警信息 + prompt} ─▶│                       │
  │                        │                       │
  │◀── Thought + ToolCall ─│                       │
  │                        │                       │
  │── Execute Tool ───────▶│──────────────────────▶│ query_logs
  │                        │                       │
  │◀── Tool Result ────────│◀──────────────────────│
  │                        │                       │
  │── {结果 + 历史} ──────▶│                       │
  │                        │                       │
  │◀── Thought + ToolCall ─│                       │
  │                        │                       │
  │── Execute Tool ───────▶│──────────────────────▶│ query_metrics
  │                        │                       │
  │◀── Tool Result ────────│◀──────────────────────│
  │                        │                       │
  │── {结果} ─────────────▶│                       │
  │                        │                       │
  │◀── Final Answer ───────│                       │
  │                        │                       │
```

---

## 6. 数据库表设计

> 使用 PostgreSQL 15+ + pgvector 扩展。所有表使用 UUID 主键，避免自增 ID 在分布式场景的问题。时间戳统一使用 `TIMESTAMPTZ`。

### 6.1 核心表

#### alerts — 告警记录表

```sql
CREATE TABLE alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_id     VARCHAR(255),                     -- 外部系统告警 ID（如 Prometheus fingerprint）
    source          VARCHAR(50)  NOT NULL,            -- alertmanager / grafana / webhook
    severity        VARCHAR(20)  NOT NULL,            -- critical / warning / info
    status          VARCHAR(20)  NOT NULL DEFAULT 'pending',  -- pending / analyzing / resolved / suppressed
    title           VARCHAR(500) NOT NULL,            -- 告警标题
    description     TEXT,                             -- 告警描述
    labels          JSONB,                            -- {"service":"api","env":"prod","instance":"x"}
    annotations     JSONB,                            -- {"summary":"...","runbook_url":"..."}
    raw_payload     JSONB,                            -- 原始 webhook 数据（调试用）
    started_at      TIMESTAMPTZ NOT NULL,             -- 告警开始时间
    resolved_at     TIMESTAMPTZ,                      -- 告警恢复时间
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_alerts_status ON alerts(status);
CREATE INDEX idx_alerts_started_at ON alerts(started_at DESC);
CREATE INDEX idx_alerts_labels ON alerts USING GIN(labels);
```

#### analysis_tasks — 分析任务表

```sql
CREATE TABLE analysis_tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_id        UUID NOT NULL REFERENCES alerts(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending / running / completed / failed
    agent_model     VARCHAR(100),                      -- 使用的 LLM 模型
    agent_iterations INT DEFAULT 0,                    -- Agent 实际迭代次数
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tasks_alert ON analysis_tasks(alert_id);
CREATE INDEX idx_tasks_status ON analysis_tasks(status);
```

#### agent_traces — Agent 推理过程记录

```sql
CREATE TABLE agent_traces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL REFERENCES analysis_tasks(id),
    iteration       INT NOT NULL,                     -- 第几轮迭代
    type            VARCHAR(20) NOT NULL,             -- thought / action / observation / final_answer
    content         TEXT NOT NULL,                    -- 内容
    tool_name       VARCHAR(100),                     -- 调用的工具名（action 类型时）
    tool_input      JSONB,                            -- 工具输入参数
    tool_output     TEXT,                             -- 工具返回结果
    token_usage     JSONB,                            -- {"prompt_tokens": N, "completion_tokens": M}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_traces_task ON agent_traces(task_id, iteration);
```

#### analysis_reports — 诊断报告表

```sql
CREATE TABLE analysis_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL UNIQUE REFERENCES analysis_tasks(id),
    alert_id        UUID NOT NULL REFERENCES alerts(id),
    root_cause      TEXT NOT NULL,                    -- 根因摘要
    confidence      DECIMAL(3,2),                     -- 置信度 0.00 ~ 1.00
    evidence_chain  JSONB,                            -- 证据链 [{"source":"logs","finding":"..."}, ...]
    affected_scope  JSONB,                            -- {"services":["api"],"metrics":["error_rate"]}
    suggestions     JSONB,                            -- ["回滚到 v1.2.3", "扩容 3→6 实例"]
    raw_report      TEXT,                             -- Agent 原始输出（Markdown）
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### historical_cases — 历史案例（RAG 检索源）

```sql
CREATE TABLE historical_cases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id       UUID NOT NULL REFERENCES analysis_reports(id),
    title           VARCHAR(500) NOT NULL,            -- 案例标题
    description     TEXT NOT NULL,                    -- 案例描述（用于 embedding 输入）
    severity        VARCHAR(20),
    service_name    VARCHAR(255),                     -- 关联服务
    tags            TEXT[],                           -- {oom, memory-leak, deployment}
    resolved_by     VARCHAR(255),                     -- 解决方式（人工标注）
    is_verified     BOOLEAN DEFAULT FALSE,            -- 是否经过人工验证（未验证的不加入检索）
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_cases_service ON historical_cases(service_name);
CREATE INDEX idx_cases_tags ON historical_cases USING GIN(tags);
```

#### case_embeddings — 案例向量表

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE case_embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    case_id         UUID NOT NULL UNIQUE REFERENCES historical_cases(id),
    embedding       vector(1536),                     -- OpenAI text-embedding-3-small 维度
    embedding_model VARCHAR(100),                     -- 使用的 embedding 模型
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- IVF 索引适合 10w+ 数据，MVP 阶段用简单索引即可
CREATE INDEX idx_embeddings_vector ON case_embeddings
    USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
```

#### datasource_configs — 数据源配置

```sql
CREATE TABLE datasource_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL UNIQUE,     -- 配置名称
    type            VARCHAR(50) NOT NULL,             -- prometheus / loki / jaeger / elasticsearch
    url             VARCHAR(500) NOT NULL,             -- 数据源 URL
    auth_type       VARCHAR(50) DEFAULT 'none',       -- none / basic / bearer
    auth_config     JSONB,                            -- {"username":"x","password":"x"} 或 {"token":"x"}
    is_enabled      BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 6.2 ER 关系

```
alerts 1 ──── 1 analysis_tasks 1 ──── N agent_traces
                │
                1 ──── 1 analysis_reports 1 ──── 1 historical_cases 1 ──── 1 case_embeddings
```

### 6.3 索引策略

| 表 | 索引 | 用途 |
|----|------|------|
| alerts | GIN on labels | 按标签筛选告警 |
| alerts | B-tree on started_at | 时间范围查询 |
| analysis_tasks | B-tree on status | 查询待处理任务 |
| case_embeddings | IVFFlat on embedding | 向量相似度搜索 |
| historical_cases | GIN on tags | 按标签检索案例 |

---

## 7. Redis 使用场景

> **MVP 阶段：不引入 Redis**，使用内存 map + PostgreSQL 替代。以下为引入 Redis 后的场景。

### 7.1 使用场景

| 场景 | Key 模式 | 数据结构 | TTL | 说明 |
|------|----------|----------|-----|------|
| 分析任务状态缓存 | `task:{task_id}:status` | String | 1h | 避免高频查询 DB |
| Agent 上下文缓存 | `agent:{task_id}:context` | List | 1h | ReAct 循环期间的对话历史 |
| Tool 调用结果缓存 | `tool:{tool_name}:{hash}` | String | 5min | 相同参数的查询结果复用 |
| 向量检索结果缓存 | `rag:{query_hash}:results` | String (JSON) | 30min | 相似告警检索缓存 |
| LLM 响应缓存 | `llm:{prompt_hash}:response` | String | 1h | 相同 prompt 不重复调用 LLM |
| 限流计数器 | `ratelimit:{ip}:{endpoint}` | String | 1min | API 调用频率限制 |
| 数据源健康状态 | `datasource:{name}:health` | String | 30s | 数据源连通性检查结果 |
| 通知去重 | `notify:{alert_id}:{channel}` | String | 5min | 防止重复通知 |

### 7.2 MVP 替代方案

```
分析任务状态   → 直接查 PostgreSQL（单条记录，性能足够）
Tool 结果缓存  → sync.Map 内存缓存
限流          → golang.org/x/time/rate 内存令牌桶
通知去重      → PostgreSQL advisory lock 或内存 map
```

**结论：MVP 阶段不安装 Redis，减少运维负担。当 QPS > 100 或需要多实例部署时再引入。**

---

## 8. RabbitMQ 使用场景

> **MVP 阶段：不引入 RabbitMQ**，使用 Go channel 内存队列。以下为引入 RabbitMQ 后的场景。

### 8.1 使用场景

| 场景 | Exchange 类型 | Queue | 说明 |
|------|---------------|-------|------|
| 分析任务分发 | Direct | `analysis.tasks` | 告警到达 → 投递分析任务 |
| 分析结果通知 | Fanout | `analysis.results` | 报告生成 → 通知各个渠道 |
| 失败重试 | Direct | `analysis.retry` | 分析失败 → 延迟重试（死信队列 + TTL） |
| Embedding 生成 | Direct | `embedding.tasks` | 新案例入库 → 异步生成 embedding |

### 8.2 MVP 替代方案

```go
// 基于 Go channel 的内存任务队列
type TaskQueue struct {
    queue chan *AnalysisTask
    // 内存队列，进程重启丢失，但任务可从 DB 恢复
}

// 启动时从 DB 恢复未完成任务
func (q *TaskQueue) RecoverPending() {
    tasks := store.GetPendingTasks()
    for _, t := range tasks {
        q.queue <- t
    }
}
```

### 8.3 何时引入 RabbitMQ

- 需要多实例部署，任务必须跨进程分发
- 需要持久化队列，保证进程重启不丢任务
- 需要复杂的路由规则（如按服务名分发到不同 Worker）

**结论：MVP 阶段用 Go channel + DB 恢复机制替代。队列容量默认 1000，超出时写入 DB 延迟处理。**

---

## 9. Agent 设计

### 9.1 ReAct Agent 核心原理

```
ReAct = Reasoning + Acting
即：模型先思考(Thought)，再行动(Action)，观察结果(Observation)，循环直到得出结论。

示例流程：

Thought: 告警显示 api-service 的 error_rate 飙升到 15%。需要先查看该服务在告警时段的错误日志。

Action: query_logs(service="api-service", time_range="last_30min", level="ERROR")

Observation: [返回 150 条日志] 其中 80% 是 "connection timeout to redis-cluster"，其余是 "circuit breaker open"。

Thought: 大量 Redis 连接超时导致了故障。需要确认 Redis 在那个时段的状态，同时查看是否有相关的变更。

Action: query_metrics(query="redis_connection_pool_active", time_range="last_30min")

Observation: Redis 连接池在 14:05 达到上限 500，之后持续饱和。

Action: search_cases(query="redis connection pool exhausted circuit breaker")

Observation: [匹配 2 个历史案例] 案例#42: Redis 连接泄漏导致池满，根因是 v2.3.1 的 connection bug。

Thought: 结合日志、指标和历史案例，根因很可能是 Redis 连接泄漏。

Final Answer: {根因分析报告}
```

### 9.2 Agent 结构

```go
// Agent 核心结构
type ReActAgent struct {
    llmClient    llm.Client           // LLM 客户端
    toolRegistry *ToolRegistry         // 工具注册表
    memory       *AgentMemory          // 上下文管理
    maxIterations int                  // 最大迭代次数（默认 10）
}

type AgentMemory struct {
    systemPrompt string
    messages     []llm.Message          // 完整对话历史
    maxTokens    int                    // 上下文窗口上限
}

// ReAct 循环
func (a *ReActAgent) Run(ctx context.Context, task *AnalysisTask) (*AgentResult, error) {
    a.memory.Reset()
    a.memory.AddSystemMessage(a.buildSystemPrompt(task))

    for i := 0; i < a.maxIterations; i++ {
        response, err := a.llmClient.Chat(ctx, a.memory.GetMessages())
        if err != nil { return nil, err }

        msg := parseResponse(response)  // 解析 Thought / Action / Final Answer

        switch msg.Type {
        case "final_answer":
            return a.buildResult(msg.Content, a.memory), nil

        case "action":
            // 执行工具调用，记录到 tracing
            tool := a.toolRegistry.Get(msg.ToolName)
            output, err := tool.Execute(ctx, msg.ToolInput)

            a.memory.AddObservation(msg.ToolName, output)
            a.recordTrace(i, msg, output)  // 持久化 agent_traces

        case "thought":
            a.memory.AddAssistantMessage(msg.Content)
        }
    }

    return nil, ErrMaxIterationsExceeded
}
```

### 9.3 System Prompt 设计

```
你是一个 SRE 智能诊断助手。你的任务是分析系统告警，找出根因。

## 你的能力
你可以使用以下工具获取信息：
- query_logs: 查询服务日志
- query_metrics: 查询 Prometheus 指标
- query_traces: 查询分布式调用链
- search_cases: 搜索历史相似案例

## 分析原则
1. 先收集信息，再做判断，不要猜测
2. 每次只调用一个工具，根据结果决定下一步
3. 优先查日志，再查指标，最后查调用链
4. 关注时间相关性：告警时段前后的数据最有价值
5. 如果找到历史相似案例，优先参考其根因和解决方案
6. 日志中看到错误 → 查相关指标
7. 指标异常 → 查相同时间段的相关服务

## 当前告警
{alert_info}

## 可用数据源
{datasource_info}

## 输出格式
当你收集到足够信息后，以 Final Answer 格式输出：
1. 根因分析（Root Cause）
2. 证据链（Evidence）
3. 影响范围（Scope）
4. 修复建议（Suggestions）
5. 置信度（Confidence: 0-100%）
```

### 9.4 关键设计决策

| 决策 | 选择 | 理由 |
|------|------|------|
| 工具调用方式 | Function Calling（原生） | 主流 LLM 均支持，解析准确 |
| 迭代上限 | 10 次 | 避免无限循环消耗 Token，实际场景 3-5 轮足够 |
| 上下文窗口 | 动态裁剪 | 超过 token 限制时，保留 system prompt + 最近 3 轮 |
| 多工具并行 | 不支持 | 简化实现，单步调试友好 |
| 失败处理 | 最多重试 2 次 | 给 LLM 反馈错误信息，让它调整参数重试 |

---

## 10. Tool Calling 设计

### 10.1 工具接口

```go
// Tool 统一接口
type Tool interface {
    Name()        string
    Description() string
    Parameters()  json.RawMessage    // JSON Schema 格式
    Execute(ctx context.Context, input json.RawMessage) (string, error)
}
```

### 10.2 工具清单

#### Tool 1: query_logs

```
名称: query_logs
描述: 查询指定服务的日志，可按时间范围、日志级别、关键字过滤。
参数:
  - service: string (必需) - 服务名称
  - time_range: string (必需) - 时间范围，如 "last_15min", "last_1h"
  - level: string (可选) - 日志级别: ERROR, WARN, INFO
  - keyword: string (可选) - 关键字搜索
  - limit: int (可选) - 返回条数，默认 100

返回: 格式化的日志条目列表（含时间戳、级别、内容）
```

#### Tool 2: query_metrics

```
名称: query_metrics
描述: 查询 Prometheus 指标，支持 PromQL 表达式或预定义查询模板。
参数:
  - query: string (必需) - PromQL 表达式 或 预定义查询名 (如 "cpu_usage", "error_rate")
  - time_range: string (必需) - 时间范围
  - step: string (可选) - 采样步长，默认 "1m"

返回: 时间序列数据点列表
```

#### Tool 3: query_traces

```
名称: query_traces
描述: 查询分布式调用链，按服务名、状态、耗时过滤。
参数:
  - service: string (必需) - 服务名称
  - time_range: string (必需) - 时间范围
  - min_duration: string (可选) - 最小耗时，如 "500ms"
  - status: string (可选) - error / ok
  - limit: int (可选) - 返回条数，默认 20

返回: 格式化的 Trace 列表（含 Span 信息、耗时、状态）
```

#### Tool 4: search_cases

```
名称: search_cases
描述: 搜索历史诊断案例，基于语义相似度匹配。
参数:
  - query: string (必需) - 搜索描述（自然语言）
  - service: string (可选) - 限制服务范围
  - top_k: int (可选) - 返回条数，默认 3

返回: 匹配的历史案例（含根因、解决方案、相似度分数）
```

### 10.3 工具注册与发现

```go
type ToolRegistry struct {
    tools map[string]Tool
}

func (r *ToolRegistry) Register(t Tool) {
    r.tools[t.Name()] = t
}

// 生成 LLM Function Calling 格式的工具定义
func (r *ToolRegistry) ToFunctionDefinitions() []llm.FunctionDef {
    var defs []llm.FunctionDef
    for _, t := range r.tools {
        defs = append(defs, llm.FunctionDef{
            Name:        t.Name(),
            Description: t.Description(),
            Parameters:  t.Parameters(),
        })
    }
    return defs
}
```

### 10.4 工具返回格式

所有工具统一返回结构化文本，便于 LLM 理解：

```
[工具名称] query_logs
[查询参数] service=api-service, time_range=last_30min, level=ERROR
[结果数量] 150 条
[数据]
2026-06-09 14:05:01 ERROR connection timeout to redis://10.0.1.5:6379 (attempt 3/3)
2026-06-09 14:05:02 ERROR circuit breaker open: redis-cache
2026-06-09 14:05:03 ERROR failed to serve request /api/users: upstream timeout
...
[前略] 按时间倒序，最多显示 20 条
```

---

## 11. RAG 设计

### 11.1 整体流程

```
┌──────────────────────────────────────────────────────────┐
│                     离线索引（异步）                       │
│                                                            │
│  诊断报告完成 → 人工确认 → 提取案例摘要 → 生成 Embedding  │
│                                                    │       │
│                                         存入 pgvector     │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                     在线检索（同步）                       │
│                                                            │
│  Agent 调用 search_cases ──► 查询文本                      │
│                        ──► embedder.Embed(query)           │
│                        ──► pgvector 余弦相似度搜索          │
│                        ──► 返回 Top-K 案例                 │
└──────────────────────────────────────────────────────────┘
```

### 11.2 Embedding 模型选择

| 模型 | 维度 | 成本 | 适用场景 |
|------|------|------|----------|
| OpenAI text-embedding-3-small | 1536 | $0.02/1M tokens | 推荐，性价比高 |
| OpenAI text-embedding-3-large | 3072 | $0.13/1M tokens | 精度要求高 |
| BGE-M3 (Ollama 本地) | 1024 | 免费 | 离线/隐私场景 |

**MVP 默认使用 OpenAI text-embedding-3-small**，可通过配置切换到 Ollama 本地模型。

### 11.3 检索策略

```go
// 混合检索：语义 + 关键词
func (r *Retriever) Search(ctx context.Context, query string, service string, topK int) ([]*Case, error) {
    // 1. 生成查询向量
    vec, _ := r.embedder.Embed(ctx, query)

    // 2. 语义检索
    sqlResults := r.store.VectorSearch(ctx, vec, topK*2)  // 召回 2x，后续重排

    // 3. 关键词过滤（服务名精确匹配加权）
    for i, c := range sqlResults {
        if c.ServiceName == service {
            sqlResults[i].Score *= 1.5  // 同服务案例加权
        }
    }

    // 4. 按相似度排序，返回 topK
    sort.Slice(sqlResults, func(i, j int) bool {
        return sqlResults[i].Score > sqlResults[j].Score
    })

    return sqlResults[:min(topK, len(sqlResults))], nil
}
```

### 11.4 案例质量保证

```
仅加入检索池的条件（AND 关系）：
  1. historical_cases.is_verified = TRUE  ← 必须人工确认过
  2. 根因置信度 confidence >= 0.6         ← 模型自评可信
  3. 案例标签不含 "false-positive"        ← 非误报

未验证的案例仅展示在 Web UI，不参与检索。
```

### 11.5 RAG 上下文注入格式

```
[历史案例参考]
以下是与当前告警相似的历史案例：

案例 #42（相似度 0.92）:
- 时间: 2026-06-01 15:00
- 服务: api-service
- 症状: Redis 连接超时 + Error Rate 飙升
- 根因: Redis client 版本 v2.3.1 存在连接泄漏 bug
- 解决: 回滚到 v2.2.8 并重启服务

案例 #17（相似度 0.78）:
- 时间: 2026-05-28 10:30
- 服务: api-service
- 症状: 数据库慢查询导致请求堆积
- 根因: 新增的 /api/reports 查询未建索引
- 解决: 添加复合索引 idx_reports_created_at_user_id

请参考以上案例，结合当前实际数据进行分析。
```

---

## 12. MVP 最小可行版本范围

### 12.1 MVP 包含

| 模块 | 功能 | 说明 |
|------|------|------|
| 告警接收 | Prometheus AlertManager Webhook | 仅支持此一种格式 |
| 告警管理 | 告警列表、状态变更 | Web UI 基础页面 |
| 数据源 | Prometheus + Loki | 仅支持这两种数据源 |
| Agent | ReAct 引擎 + 2 个 Tool | query_logs + query_metrics |
| LLM | OpenAI 兼容 API | 支持 OpenAI / Ollama |
| 报告 | 结构化 Markdown 输出 | 不做前端富文本渲染 |
| 存储 | PostgreSQL（含 pgvector） | 唯一必需依赖 |
| 通知 | 飞书 Webhook | 仅一种通知渠道 |
| Web UI | 告警列表 + 报告查看 | 单页应用，最小化 |

### 12.2 MVP 不包含

| 功能 | 说明 | 计划 |
|------|------|------|
| 调用链分析 (query_traces) | 需要 Jaeger/Tempo 集成 | v1.1 |
| RAG 历史案例检索 | 需要积累案例 + embedding | v1.2 |
| 多源告警格式 | 仅 AlertManager | v1.1 加 Grafana |
| Redis / RabbitMQ | 内存替代 | v1.3+ |
| 多租户 | 单团队使用 | v2.0 |
| 告警聚合/降噪 | 直接透传 | v1.1 |
| 自动修复 | 仅输出建议 | v2.0 |
| 仪表盘 | 无 | v1.2 |

### 12.3 MVP 技术栈（最小化）

```
Go 1.22+                          # 后端
Gin                               # HTTP 框架
PostgreSQL 15+ + pgvector         # 数据库 + 向量存储
OpenAI 兼容 SDK                   # LLM 调用
Vue 3 + 朴素 HTML                 # 前端（或直接用 Go template 渲染）
Docker Compose                    # 部署（3 个容器：app + pg + optional-ollama）
```

### 12.4 MVP 开发顺序

```
Phase 1（2-3 天）: 项目骨架
  - Go 项目初始化、配置加载、数据库迁移
  - 告警接收 API（AlertManager webhook）

Phase 2（2-3 天）: 数据源集成
  - Prometheus 查询适配器
  - Loki 查询适配器

Phase 3（3-5 天）: Agent 引擎
  - LLM 客户端封装
  - ReAct 循环引擎
  - query_logs / query_metrics Tool 实现
  - System Prompt 调优

Phase 4（2-3 天）: 报告与通知
  - 结构化报告生成
  - agent_traces 持久化
  - 飞书通知

Phase 5（2-3 天）: Web UI
  - 告警列表页
  - 报告详情页
  - 手动触发分析

总计: 11-17 天（个人开发者）
```

### 12.5 MVP 配置文件示例

```yaml
# config.yaml — MVP 最小配置
server:
  port: 8080

database:
  url: "postgres://alertmind:password@localhost:5432/alertmind?sslmode=disable"

llm:
  provider: "openai"              # openai / ollama
  api_key: "${OPENAI_API_KEY}"
  base_url: "https://api.openai.com/v1"  # Ollama 则改为 http://localhost:11434/v1
  model: "gpt-4o-mini"            # 或 qwen2.5:7b (Ollama)
  embedding_model: "text-embedding-3-small"

datasources:
  - name: "main-prometheus"
    type: "prometheus"
    url: "http://prometheus:9090"

  - name: "main-loki"
    type: "loki"
    url: "http://loki:3100"

alert:
  sources:
    - type: "alertmanager"

notification:
  feishu_webhook: "${FEISHU_WEBHOOK_URL}"
```

---

## 13. 技术选型总览

| 层级 | 技术 | 用途 | 必要性 | 备注 |
|------|------|------|--------|------|
| 语言 | Go 1.22+ | 后端 | 必须 | 单二进制，性能好 |
| HTTP | Gin | API 路由 | 必须 | 最常用 Go Web 框架 |
| 数据库 | PostgreSQL 15+ | 持久化 | 必须 | 唯一必需依赖 |
| 向量 | pgvector | embedding 存储 | MVP 可选 | 为 RAG 做准备 |
| LLM | OpenAI 兼容 API | AI 分析 | 必须 | 可换 Ollama 本地模型 |
| 前端 | Vue 3 或 Go template | Web UI | 必须 | MVP 建议先 Go template |
| 通知 | 飞书 Webhook | 告警通知 | 必须 | 最简通知方式 |
| 部署 | Docker Compose | 容器化 | 推荐 | 方便部署 |

**刻意不引入的技术：**

| 技术 | 不引入原因 |
|------|------------|
| Redis | MVP 用内存缓存 + DB 替代 |
| RabbitMQ/Kafka | MVP 用 Go channel 内存队列替代 |
| Elasticsearch | 用 Loki 替代日志存储（运维更轻） |
| gRPC | 单体应用无需 RPC |
| K8s | 单体应用 Docker Compose 足够 |
| LangChain/LangChainGo | Go 版不成熟，自己实现量更小 |
| 专门的向量 DB (Milvus/Qdrant) | pgvector 够用，少一个运维负担 |

---

## 14. 项目目录结构

```
alert-mind/
├── cmd/
│   └── server/
│       └── main.go                 # 入口：启动 HTTP + Worker
│
├── internal/
│   ├── alert/
│   │   ├── receiver.go             # Webhook 接收
│   │   ├── parser.go               # AlertManager 格式解析
│   │   └── normalizer.go           # → 标准化 Alert 结构
│   │
│   ├── collector/
│   │   ├── collector.go            # DataCollector 接口定义
│   │   ├── prometheus.go           # PromQL 查询实现
│   │   └── loki.go                 # LogQL 查询实现
│   │
│   ├── agent/
│   │   ├── engine.go               # ReAct 主循环
│   │   ├── prompt.go               # System Prompt 构建
│   │   └── registry.go             # ToolRegistry
│   │
│   ├── tools/
│   │   ├── tool.go                 # Tool 接口
│   │   ├── query_logs.go
│   │   └── query_metrics.go
│   │
│   ├── llm/
│   │   ├── client.go               # OpenAI 兼容 HTTP Client
│   │   └── types.go                # ChatMessage, FunctionDef 等
│   │
│   ├── report/
│   │   ├── generator.go            # 报告结构提取
│   │   └── notifier.go             # 飞书通知
│   │
│   └── store/
│       ├── db.go                   # PostgreSQL 连接管理
│       ├── alert_repo.go
│       ├── task_repo.go
│       ├── report_repo.go
│       └── migrations/
│           ├── 001_alerts.sql
│           ├── 002_tasks.sql
│           ├── 003_traces.sql
│           ├── 004_reports.sql
│           ├── 005_datasources.sql
│           └── 006_rag.sql         # cases + embeddings（v1.2 启用）
│
├── config/
│   ├── config.go                   # Viper 配置加载
│   └── config.example.yaml         # 配置模板
│
├── web/
│   └── templates/
│       ├── alerts.html             # 告警列表页
│       └── report.html             # 报告详情页
│
├── docker-compose.yml              # app + postgres
├── Dockerfile
├── Makefile
├── go.mod
├── go.sum
└── DESIGN.md                       # 本文档
```

---

## 附录 A: MVP 核心接口定义

```go
// --- 告警相关 ---
type Alert struct {
    ID          string            `json:"id"`
    ExternalID  string            `json:"external_id"`
    Source      string            `json:"source"`
    Severity    string            `json:"severity"`
    Status      string            `json:"status"`
    Title       string            `json:"title"`
    Description string            `json:"description"`
    Labels      map[string]string `json:"labels"`
    Annotations map[string]string `json:"annotations"`
    StartedAt   time.Time         `json:"started_at"`
}

// --- Agent 相关 ---
type AgentResult struct {
    RootCause     string   `json:"root_cause"`
    EvidenceChain []Evidence `json:"evidence_chain"`
    AffectedScope []string `json:"affected_scope"`
    Suggestions   []string `json:"suggestions"`
    Confidence    float64  `json:"confidence"`
    Iterations    int      `json:"iterations"`
    TokenUsage    int      `json:"token_usage"`
}

type Evidence struct {
    Source string `json:"source"` // logs / metrics / traces / cases
    Finding string `json:"finding"`
    Data    string `json:"data"` // 原始数据片段
}

// --- Tool 相关 ---
type ToolCall struct {
    Name      string          `json:"name"`
    Input     json.RawMessage `json:"input"`
}

type ToolResult struct {
    Success bool   `json:"success"`
    Output  string `json:"output"`
    Error   string `json:"error,omitempty"`
}
```

---

## 附录 B: MVP docker-compose.yml

```yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://alertmind:alertmind@postgres:5432/alertmind?sslmode=disable
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - FEISHU_WEBHOOK_URL=${FEISHU_WEBHOOK_URL}
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_USER: alertmind
      POSTGRES_PASSWORD: alertmind
      POSTGRES_DB: alertmind
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U alertmind"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```
