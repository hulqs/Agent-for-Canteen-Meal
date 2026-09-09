# Tool · `canteen-menu-query` 食堂菜单查询 ★标杆工具

> ⚠️ **迁移声明**：原位于 `skills/canteen-menu-query/`，经 skill↔tool 评审（见 `docs/00-项目结构.md §3`）判定为"确定性执行、显式参数、结构化返回、外部交互"的工具能力，已迁入 `agent/tools/`。文档内容保持兼容，沿用原技能契约与容错设计。

> **这是本项目的标杆工具（Tool）**，完整演示了"独立 HTTP 服务 + 标准 JSON 入出参 + 被主智能体自动调用 + 完整容错"四项要求。新增其他工具请直接复制本 README 的结构。

| 项 | 值 |
| --- | --- |
> 编号 **T1**（见 `docs/05-技能清单与接口契约.md`）。

| 名称 | `canteen-menu-query` |
| 版本 | `1.2.0` |
| 端口 | `8201`（工具段 8200+） |
| 类型 | 只读查询 |
| 依赖 | 上游食堂菜品系统 API / 菜单数据库、Redis（缓存）、知识库枚举字典（食堂/楼层/餐次/辣度） |
| SLA | P95 ≤ 500ms，可用性 ≥ 99.5% |
| 被调用方 | 主智能体（ReAct `tool_call`）、S1 `dish-recommender`（skill→tool，强依赖） |

---

## 1. 它做什么

查询校园食堂的**真实**菜单数据：某天、某餐次、某食堂/窗口供应哪些菜，含价格、辣度、供应时段、特色描述、过敏原标签、余量状态、核验日期与营业时间。

**不做**：营养估算（→ `nutrition-analyzer`）、排队预测（→ `crowd-forecast`）、组合推荐（→ `dish-recommender`）、天气（→ `weather-query`）。

---

## 2. 快速契约

```
POST http://canteen-menu-query:8201/v1/{action}
GET  http://canteen-menu-query:8201/health
GET  http://canteen-menu-query:8201/openapi.json
GET  http://canteen-menu-query:8201/metrics
```

| action | 说明 |
| --- | --- |
| `query_dishes` | 按条件查菜品列表 |
| `get_dish_detail` | 单菜品详情（配料、过敏原、可能交叉污染、核验日期） |
| `get_canteen_info` | 食堂/窗口信息与营业时间（**权威源：知识库 `03_营业与服务/营业时间.md`**） |
| `check_availability` | 指定菜品余量 |

完整字段定义见 [docs/05-技能清单与接口契约.md](../../docs/05-技能清单与接口契约.md) 的 **T1** 章节；机器可读契约见 `openapi.yaml`（实现阶段产出）。

---

## 3. 请求示例

```json
{
  "request_id": "req_01J9Z8X7K2M3N4P5Q6R7S8T9V0",
  "skill": "canteen-menu-query",
  "version": "1.2.0",
  "action": "query_dishes",
  "params": {
    "campus": "主校区",
    "canteen": "一食堂",
    "floor": 2,
    "date": "2026-09-08",
    "meal_period": "lunch",
    "keywords": ["鸡肉"],
    "filters": {
      "price_max": 15,
      "spice_level_max": 1,
      "tags_include": ["高蛋白"],
      "allergen_exclude": ["花生"],
      "available_only": true
    },
    "sort": { "field": "price", "order": "asc" },
    "pagination": { "page": 1, "page_size": 20 }
  },
  "context": { "user_id_hash": "u_9f2c1a", "trace_id": "tr_01J9Z", "deadline_ms": 800 }
}
```

**参数约束**

| 参数 | 类型 | 必填 | 校验规则 |
| --- | --- | --- | --- |
| `campus` | string | ❌ | 校区白名单（来自知识库 `01_食堂基础信息`「所在校区」，当前待填写）；缺省取会话 `campus_hint`；非法值 `40001` |
| `canteen` | string | ❌ | 枚举 `一食堂`/`二食堂`/`三食堂`；别名（第一食堂/1 食堂/一餐…）由归一化层映射，见 `docs/08 §1.4`；非法值 `40001` |
| `floor` | int | ❌ | 枚举 `1`/`2`（知识库「楼层布局」）；越界 `40001` |
| `date` | string(date) | ❌ | 缺省=今天；范围 [今天, +7 天]，越界返回 `40001` |
| `meal_period` | enum | ❌ | `breakfast`/`lunch`/`dinner`（对应知识库早/午/晚）；`night_snack` 保留但**知识库无来源**，命中不得臆造；缺省按当前时间推断 |
| `keywords` | string[] | ❌ | 最多 5 个，单条 ≤ 20 字 |
| `filters.price_max` | number | ❌ | 0 < x ≤ 200 |
| `filters.spice_level_max` | int | ❌ | 0–3（0 不辣 / 1 微辣 / 2 中辣 / 3 特辣，知识库中文辣度入库时归一化） |
| `filters.allergen_exclude` | string[] | ❌ | 枚举：花生/海鲜/牛奶/鸡蛋/麸质/大豆/坚果；**硬过滤** |
| `filters.available_only` | bool | ❌ | 默认 true |
| `pagination.page_size` | int | ❌ | 1–50，默认 20 |

---

## 4. 响应示例

### 4.1 成功（实时）

```json
{
  "request_id": "req_01J9Z8X7K2M3N4P5Q6R7S8T9V0",
  "code": 0,
  "message": "ok",
  "data": {
    "canteen": {
      "campus": "主校区", "name": "一食堂", "floor": 2,
      "open_hours": { "lunch": "10:30-13:00" }, "status": "open",
      "hours_source": "03_营业与服务/营业时间.md"
    },
    "dishes": [
      {
        "dish_id": "d_10087", "name": "香煎鸡胸饭", "window": "2F-轻食窗口",
        "price": 12.0, "spice_level": 0, "supply_period": "午餐",
        "feature_desc": "低油煎制，配时蔬",
        "tags": ["高蛋白", "低脂"],
        "allergens": [], "may_contain": ["大豆"],
        "availability": "in_stock",
        "verified_date": "2026-09-08", "updated_at": "2026-09-08T11:32:00+08:00"
      }
    ],
    "total": 3, "page": 1
  },
  "meta": {
    "latency_ms": 86, "skill_version": "1.2.0",
    "data_version": "menu_2026-09-08_v3", "source": "upstream_api",
    "degraded": false, "cache_hit": false,
    "warnings": ["已为你排除含花生及可能含花生的菜品"]
  }
}

> `verified_date`（最后核验日期）是知识库规范要求的必带字段；缺失时主智能体不得把价格当已核实事实用，应走"暂未查到，建议到窗口确认"。
```

### 4.2 降级（上游超时 → 缓存）

```json
{
  "request_id": "req_...",
  "code": 0,
  "message": "ok（数据来自缓存，可能非实时）",
  "data": { "dishes": ["..."], "total": 3 },
  "meta": {
    "latency_ms": 812, "source": "cache", "degraded": true, "cache_hit": true,
    "data_version": "menu_2026-09-08_v3", "cached_at": "2026-09-08T11:42:10+08:00",
    "warnings": ["上游菜单服务超时，当前为 60 秒前缓存，实际以窗口现场为准"]
  }
}
```

### 4.3 无数据

```json
{
  "request_id": "req_...",
  "code": 40404,
  "message": "当前条件下没有匹配的菜品。建议：放宽预算至 12 元以上，或去掉'高蛋白'标签；也可查看一食堂其他窗口。",
  "data": { "suggestions": { "min_feasible_price": 12.0, "relaxed_filters": ["price_max:15→18"] } },
  "meta": { "degraded": false, "source": "upstream_api" }
}
```

### 4.4 完全失败（兜底）

```json
{
  "request_id": "req_...",
  "code": 50420,
  "message": "菜单服务暂时不可用，已尝试缓存与快照均未命中。建议直接到食堂窗口查看，或稍后再问。",
  "data": null,
  "meta": { "latency_ms": 950, "degraded": true, "source": "none" }
}
```

---

## 5. 容错设计（重点）

### 5.1 降级链

| 级别 | 数据源 | 触发条件 | `meta.source` | 用户可见提示 |
| --- | --- | --- | --- | --- |
| L0 | 上游菜单 API | 正常 | `upstream_api` | 无 |
| L1 | Redis 缓存（TTL 60s） | 上游超时 / 5xx | `cache` | "数据为 60 秒前" |
| L2 | 昨日同餐次快照 | 缓存未命中 | `snapshot` | "参考昨日菜单，今日可能有调整" |
| L3 | PostgreSQL 向量召回（仅 `status=published`） | 快照缺失 | `vector_recall` | "以下为历史/文档信息，可能不准确" |
| L4 | 兜底话术 | 全部失败 | `none` | `code=50420`，话术可直接念给用户 |

### 5.2 参数配置（`config.yaml`）

| 项 | 值 |
| --- | --- |
| 连接超时 | 300ms |
| 读取超时 | 800ms（取 `min(自身, context.deadline_ms)`） |
| 重试次数 | 2（指数退避 100ms → 300ms + 抖动） |
| 重试条件 | 仅**幂等读**动作，且错误为超时/5xx；`40001/40404/40301` 不重试 |
| 熔断 | 10s 窗口，请求数 ≥ 20 且错误率 > 50% → OPEN 30s → HALF-OPEN 探测 5 次 |
| 缓存 TTL | 热点（今日菜单）60s；空结果 10s（防穿透） |
| 限流 | 令牌桶 200 QPS；超出 `42900` + `Retry-After` |
| 并发 | 单实例最大并发 100；上游连接池 50 |

### 5.3 脏数据处理

上游返回的菜品若校验失败（价格缺失、菜名为空、枚举越界），**只丢弃该条**并在 `meta.warnings` 追加"N 条数据异常已忽略"，整体仍返回可用数据——部分可用优于全盘失败。

### 5.4 幂等与只读

本工具为纯读操作，天然幂等，可安全重试。不涉及任何写操作。

> **被 S1 强依赖**：S1 `dish-recommender` 通过 HTTP 调用本工具取候选（skill→tool）。本工具若进入 L3/L4 降级，S1 会随之置自身 `degraded=true` 并提示"参考历史数据"；若本工具完全不可用（`50420`/`50300`），S1 返回 `50420` 兜底话术。

---

## 6. 被主智能体自动调度

本工具**不写死任何路由规则**。它把自己注册到 Tool Registry，由主智能体在 ReAct 循环中通过 `tool_call` 自主决定是否调用。

### 6.1 给 LLM 的工具描述（决定调度准确率的 80%）

```yaml
name: canteen_menu_query
description: >
  查询校园食堂的**真实**菜单数据：某天、某餐次、某食堂/窗口供应哪些菜，
  含价格、辣度、供应时段、特色描述、过敏原标签、余量状态、核验日期与营业时间。
  当用户问"今天/明天/中午/晚上吃什么、有什么菜、多少钱、还有没有、几点开门"时使用。
  返回的数据带 data_version、updated_at 与 verified_date，回答时应告知用户数据时间；
  若 meta.degraded 为 true，必须提示"以现场为准"；若 verified_date 缺失，按"暂未查到"处理。
  不要用于：营养/热量计算（用 nutrition_analyzer）、排队人流（用 crowd_forecast）、
  组合搭配推荐（用 dish_recommender）、天气（用 weather_query）。

when_to_use:
  - "今天一食堂中午有啥？"
  - "二楼的红烧肉还有吗？多少钱？"
  - "15 块以内能吃啥？"
  - "食堂几点开门？周末开吗？"

when_not_to_use:
  - "帮我算一下这顿多少卡"  → nutrition_analyzer
  - "哪个窗口人少不用排队"  → crowd_forecast
  - "给我搭配一个减脂午餐"  → dish_recommender
  - "今天下雨吗"            → weather_query

examples:
  - q: "二楼还有没有红烧肉"
    params: { canteen: "一食堂", floor: 2, keywords: ["红烧肉"], filters: { available_only: true } }
  - q: "明天中午 15 块以内有啥"
    params: { date: "<+1天>", meal_period: "lunch", filters: { price_max: 15, available_only: true } }
  - q: "我对花生过敏，能吃啥"
    params: { filters: { allergen_exclude: ["花生"], available_only: true } }
```

### 6.2 典型调用链

```
用户："明天中午一食堂有没有不辣的鸡胸肉，15 块以内？我对花生过敏。"
  → Planner：需要真实菜单 → 选 T1 canteen_menu_query
  → 参数生成：date=+1, meal_period=lunch, canteen=一食堂,
              keywords=[鸡胸肉], filters={price_max:15, spice_level_max:0, allergen_exclude:[花生]}
  → invoker：超时 800ms → 上游超时 → 重试 1 次 → 命中 L1 缓存
  → Observation（结构化）：status=DEGRADED source=cache degraded=true 命中 3 道菜…
  → ReAct 第 2 轮：信息充分 → finish
  → 出站护栏：因 degraded=true 与 ALLERGY 标记，强制追加两条风险提示
```

---

## 7. 错误码

| code | 含义 | 主智能体动作 |
| --- | --- | --- |
| `0` | 成功 | 正常使用 |
| `40001` | 参数校验失败（含 `date` 越界、枚举非法） | 修正参数重试 1 次 |
| `40002` | 未知 action | 改用合法 action |
| `40003` | 版本不兼容 | 不重试，告警 |
| `40301` | 越权（请求了无权限校区的数据） | 拒绝并说明 |
| `40404` | 无匹配数据 | 依据 `data.suggestions` 放宽条件或告知用户 |
| `42900` | 限流 | 退避后重试 1 次 |
| `50010` | 技能内部错误 | 换工具或降级 |
| `50410` | 上游超时 | 由技能内部已降级；仍失败则主智能体改走 RAG |
| `50420` | 上游不可用（四级降级全部失败） | 使用 `message` 兜底话术 |
| `50300` | 熔断中 | 直接降级，不重试 |

---

## 8. 可观测性

| 项 | 说明 |
| --- | --- |
| 日志 | 结构化 JSON，必带 `trace_id`、`request_id`、`action`、`code`、`latency_ms`、`source` |
| 指标 | `skill_calls_total{action,code}`、`skill_latency_seconds`、`skill_degraded_total{level}`、`skill_circuit_state`、`skill_cache_hit_ratio` |
| 健康检查 | `/health` 返回自身与上游连通性（上游不健康时仍返回 200，仅标记 `upstream: "down"`，避免被误摘除） |
| 追踪 | 透传 `X-Trace-Id`，接入 OpenTelemetry |

---

## 9. 测试

| 类型 | 用例 |
| --- | --- |
| 契约测试 | 4 个 action ×（正常 / 边界 / 非法参数） |
| 业务测试 | 过敏原硬过滤、预算过滤、辣度过滤、分页、排序、餐次推断 |
| 故障注入 | 上游超时（→ L1 缓存）、上游 500（→ L1）、缓存未命中（→ L2 快照）、Redis 宕机（→ L2/L3）、全链路失败（→ L4 兜底话术）、限流（→ 42900） |
| 性能测试 | 200 QPS 持续 5 分钟，P95 ≤ 500ms，错误率 < 1% |
| 契约一致性 | CI 校验 `openapi.yaml` 与 Pydantic Schema 一致 |

---

## 10. 目录结构（实现阶段）

```
agent/tools/canteen-menu-query/
├── README.md            ✅ 本文档
├── openapi.yaml         🔲 OpenAPI 3.1 契约
├── config.yaml          🔲 超时/重试/熔断/缓存配置
├── Dockerfile           🔲
├── requirements.txt     🔲
├── src/
│   ├── main.py          🔲 FastAPI 应用与路由
│   ├── schemas.py       🔲 入出参 Pydantic 模型
│   ├── service.py       🔲 业务逻辑与过滤
│   ├── adapters/
│   │   ├── upstream.py  🔲 上游菜单 API 适配器
│   │   ├── cache.py     🔲 Redis 缓存
│   │   ├── snapshot.py  🔲 昨日快照
│   │   └── vector_recall.py  🔲 PostgreSQL 向量召回兜底
│   ├── resilience.py    🔲 超时/重试/熔断/降级链
│   └── errors.py        🔲 错误码
└── tests/               🔲 契约测试 + 故障注入测试
```
