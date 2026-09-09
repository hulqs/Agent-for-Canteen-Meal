# Tool · `weather-query` 校园天气查询（实况 / 预报 / 官方预警）

> 按 `docs/00-项目结构.md §3` 的四维判定，天气查询属"确定性执行、显式参数、结构化返回、外部交互"的能力，归类为 **Tool**，端口段 `8200+`（本工具 `8204`）。新增工具请复制 `agent/tools/_template`。

| 项 | 值 |
| --- | --- |
| 名称 | `weather-query` |
| 版本 | `0.1.0` |
| 端口 | `8204` |
| 类型 | 只读（外部数据查询） |
| 依赖 | 外部公共气象服务（HTTP）、Redis（缓存 TTL 600s）、校区坐标配置、本地季节气候基线 |
| SLA | P95 ≤ 400ms，可用性 ≥ 99.5% |
| 是否需人工确认 | 否 |

---

## 1. 它做什么

按**校区（或授权坐标）+ 时间**查询天气实况、短时/多日预报与官方气象预警，并输出**规则化派生**的就餐出行提示（是否带伞、是否就近就餐、路面舒适度）。

**不做**：
- 不推荐具体菜品（→ `dish-recommender`，本工具只提供天气事实与规则提示）；
- 不查菜单与营业时间（→ `canteen-menu-query`）；
- 不提供医疗建议（中暑/冻伤等 → 由出站护栏注入就医指引）；
- 不自行发布或改写气象预警（只原样引用气象台口径）。

---

## 2. 契约

```
POST http://weather-query:8204/v1/{action}
GET  http://weather-query:8204/health
GET  http://weather-query:8204/openapi.json
GET  http://weather-query:8204/metrics
```

| action | 说明 |
| --- | --- |
| `get_current` | 当前实况（天气现象、气温、体感、湿度、风力、降水概率、AQI、UV） |
| `get_forecast` | 未来 1–72 小时或 1–7 天预报；支持按餐次（早/午/晚）聚合切片 |
| `get_alerts` | 官方气象预警（暴雨/高温/大风/寒潮等） |

---

## 3. 请求示例（action=`get_forecast`）

```json
{
  "request_id": "req_01J9Z8X7K2M3N4P5Q6R7S8T9V3",
  "skill": "weather-query",
  "version": "0.1.0",
  "action": "get_forecast",
  "params": {
    "campus": "主校区",
    "date": "2026-09-09",
    "meal_periods": ["lunch", "dinner"],
    "hourly_hours": 24
  },
  "context": { "user_id_hash": "u_9f2c1a", "trace_id": "tr_01J9Z", "deadline_ms": 800 }
}
```

**参数约束**

| 参数 | 类型 | 必填 | 校验规则 |
| --- | --- | --- | --- |
| `campus` | string | ❌ | 校区白名单，与 T1 共用同一枚举字典（来自知识库 `01_食堂基础信息`「所在校区」字段，当前待填写）；内部映射为预设经纬度；缺省取会话 `campus_hint`；非法值返回 `40001` |
| `location` | `{lng,lat}` | ❌ | 显式坐标；**仅在用户显式授权定位时使用**，不落画像、不落日志；与 `campus` 互斥，同时传以 `location` 为准 |
| `date` | string(date) | ❌ | 缺省 = 今天；范围 [今天, +7 天]，越界返回 `40001` |
| `meal_periods` | enum[] | ❌ | `breakfast` / `lunch` / `dinner`，返回按餐次聚合的天气切片；长度 ≤ 3 |
| `hourly_hours` | int | ❌ | 1–72，默认 24 |
| `daily_days` | int | ❌ | 1–7，默认 3；与 `hourly_hours` 同时传时以 `hourly_hours` 优先 |

---

## 4. 响应示例

### 4.1 成功（命中缓存）

```json
{
  "request_id": "req_01J9Z8X7K2M3N4P5Q6R7S8T9V3",
  "code": 0,
  "message": "ok",
  "data": {
    "location": { "campus": "主校区", "lat": 30.00, "lng": 120.00, "source": "campus_config" },
    "issued_at": "2026-09-09T15:00:00+08:00",
    "current": {
      "weather_text": "小雨", "weather_code": "rain_light",
      "temp_c": 27.4, "feels_like_c": 30.1, "humidity": 78,
      "wind_level": 3, "precip_probability": 0.6, "aqi": 62, "uv_index": 4
    },
    "meal_slices": [
      { "date": "2026-09-09", "meal_period": "lunch", "weather_text": "小雨",
        "temp_c": 28.0, "precip_probability": 0.7, "walk_comfort": "poor" },
      { "date": "2026-09-09", "meal_period": "dinner", "weather_text": "阴",
        "temp_c": 24.5, "precip_probability": 0.3, "walk_comfort": "fair" }
    ],
    "dining_advice": {
      "level": "umbrella",
      "tags": ["带伞", "就近食堂", "避开露天排队"],
      "text": "午间有小雨（降水概率 70%），建议带伞并优先选离教学楼最近的食堂。"
    },
    "alerts": []
  },
  "meta": {
    "latency_ms": 120, "skill_version": "0.1.0",
    "data_version": "wx_2026-09-09T15:00+08:00",
    "source": "cache", "degraded": false, "cache_hit": true,
    "provider": "<气象服务名>", "warnings": []
  }
}
```

### 4.2 官方预警

```json
{
  "request_id": "req_...",
  "code": 0,
  "data": {
    "alerts": [
      { "alert_id": "wx_alert_20260909_001", "type": "rainstorm", "level": "orange",
        "title": "暴雨橙色预警", "effective_at": "2026-09-09T14:00:00+08:00",
        "expire_at": "2026-09-09T22:00:00+08:00",
        "source": "当地气象台", "source_url": "https://…",
        "advice_text": "请关注学校官方通知，减少不必要外出。" }
    ]
  },
  "meta": { "source": "upstream_api", "degraded": false,
            "warnings": ["预警信息以气象台与学校官方发布为准"] }
}
```

### 4.3 降级（上游超时 → 最近一次结果）

```json
{
  "request_id": "req_...",
  "code": 0,
  "message": "ok（为 90 分钟前的最近一次结果）",
  "data": { "current": { "...": "..." }, "meal_slices": [] },
  "meta": {
    "latency_ms": 780, "source": "last_known", "degraded": true, "cache_hit": false,
    "data_version": "wx_2026-09-09T13:30+08:00",
    "warnings": ["天气服务暂时取不到最新数据，以下为最近一次结果，仅供参考"]
  }
}
```

### 4.4 完全失败（兜底）

```json
{
  "request_id": "req_...",
  "code": 50420,
  "message": "暂时查不到天气信息，建议看下窗外或直接留意学校通知～",
  "data": null,
  "meta": { "latency_ms": 900, "degraded": true, "source": "none" }
}
```

---

## 5. `dining_advice` 规则映射（确定性，可测试）

| 触发条件（按优先级命中即停） | `level` | `tags` |
| --- | --- | --- |
| 命中官方预警（暴雨/大风/寒潮/高温橙色及以上） | `alert` | 以官方通知为准 |
| `precip_probability ≥ 0.5` 或 `weather_code ∈ {rain_*}` | `umbrella` | 带伞 / 就近食堂 / 避开露天排队 |
| `temp_c ≥ 32` 或 `feels_like_c ≥ 35` | `heat` | 补水 / 清淡 / 避开正午外出 |
| `temp_c ≤ 5` 或 `feels_like_c ≤ 2` | `cold` | 热食 / 热汤 / 注意保暖 |
| `wind_level ≥ 6` | `wind` | 注意高空坠物 / 就近就餐 |
| `aqi > 150` | `aqi` | 减少户外停留 |
| 其他 | `normal` | 天气适宜 |

> 该映射是**纯函数**，与模型无关，便于单元测试与回归。它只输出环境层面的提示，不指定菜品；涉及"吃什么"必须由 `dish-recommender` 完成。

---

## 6. 容错设计

### 6.1 降级链

| 级别 | 数据源 | 触发条件 | `meta.source` | 用户可见提示 |
| --- | --- | --- | --- | --- |
| L0 | 外部气象 API | 正常 | `upstream_api` | 无 |
| L1 | Redis 缓存（TTL 600s） | 超时 / 5xx / 限流 | `cache` | "数据更新于 {time}" |
| L2 | 最近一次成功结果（≤2h） | 缓存未命中 | `last_known` | "为最近一次结果，可能已过时" |
| L3 | 本地季节气候基线（同月典型气温/降水） | 无历史 | `climatology` | "为该月份常年气候参考，非实况" |
| L4 | 兜底话术 | 全部失败 | `none` | `code=50420` |

### 6.2 参数配置（`config.yaml`）

| 项 | 值 |
| --- | --- |
| 连接超时 / 读取超时 | 300ms / 800ms（取 `min(自身, context.deadline_ms)`） |
| 重试次数 | 2（指数退避 100ms → 300ms + 抖动），纯读幂等 |
| 重试条件 | 超时 / 5xx；`40001 / 40404 / 42900` 不重试 |
| 熔断 | 10s 窗口，请求数 ≥ 20 且错误率 > 50% → OPEN 30s → HALF-OPEN 探测 5 次 |
| 缓存 TTL | 600s（实况）/ 1800s（多日预报）；空结果 60s 防穿透 |
| 限流 | 单实例令牌桶 50 QPS，超出走 L1 |
| 并发 | 单实例最大并发 50；上游连接池 20 |

### 6.3 被依赖时的非强依赖约定

`crowd-forecast`（T2）与 `dish-recommender`（S1）可把天气作为**排序/预测特征**。约定：

- 调用方传 `with_weather=true` / `context_features.weather` 才会触发；
- 本工具任何级别降级或失败时，调用方**直接退化为无天气基线**，置 `weather_used=false` 并在 `meta.warnings` 追加 `weather_feature_missing`；
- **调用方不得因天气不可用而置自身 `degraded=true` 或整体失败**（见 `docs/05 §1.1 D5`）。

---

## 7. 被主智能体自动调度

本工具**不写死任何路由规则**，注册到 Tool Registry 后由主智能体在 ReAct 循环中自主选择。

### 7.1 给 LLM 的工具描述

```yaml
name: weather_query
description: >
  查询校区当前及未来的天气实况、按餐次（早/午/晚）的天气预报与官方气象预警，
  并给出是否需要带伞、是否就近就餐等规则化出行提示。
  当用户问"下雨吗""热不热""要带伞吗""明天中午天气怎么样""这种天去食堂方便吗"时使用。
  回答时必须带上数据时间（issued_at）；若 meta.degraded 为 true，要说明数据非实时。
  本工具不推荐菜品——"这种天吃什么"要先查天气再交给 dish_recommender 搭配。
  官方预警必须原样引用，不得改写等级或自行发布。

when_to_use:
  - "今天下雨吗，去食堂要带伞吗？"
  - "明天中午热不热？"
  - "现在外面什么天气？"
  - "今天有暴雨预警吗？"

when_not_to_use:
  - "这种天吃点啥好" → 先用 weather_query 取天气特征，再交给 dish_recommender 出组合
  - "今天一食堂有什么菜" → canteen_menu_query
  - "现在哪个窗口人少" → crowd_forecast（它内部会自行决定是否用天气特征）

examples:
  - q: "今天中午去食堂要带伞吗"
    params: { campus: "主校区", date: "<今天>", meal_periods: ["lunch"] }
  - q: "明天早上冷不冷"
    params: { date: "<+1天>", meal_periods: ["breakfast"] }
  - q: "今天有什么天气预警吗"
    action: "get_alerts"
    params: { campus: "主校区" }
```

### 7.2 典型调用链

```
用户："今天中午下雨吗？一食堂吃点啥好，15 块以内。"
  → Planner：两件事——天气 + 菜单 → 并行发起 tool_call
  → [tool_call] weather_query.get_forecast(campus=主校区, date=今天, meal_periods=[lunch])
  → [tool_call] canteen_menu_query.query_dishes(canteen=一食堂, meal_period=lunch, filters={price_max:15})
  → observe：天气=小雨 pop=0.7 walk_comfort=poor；菜单命中 5 道
  → ReAct 第 2 轮：需要"搭配"而非"列举" → skill_call dish_recommender
       params.context_features.weather = "rain_light"（复用第 1 轮结果，不重复调用天气）
  → S1 内部强依赖 T1（候选已在第 1 轮取过，走缓存），可选特征 weather 生效
  → 出站护栏：附天气数据时间 + "以现场为准" + 若命中预警则追加"以学校官方通知为准"
```

---

## 8. 错误码

| code | 含义 | 主智能体动作 |
| --- | --- | --- |
| `0` | 成功 | 正常使用，标注 `issued_at` |
| `40001` | 参数不合法（日期越界 / 校区非法 / 坐标越界） | 修正后重试 1 次 |
| `40002` | 未知 action | 改用合法 action |
| `40101` | 气象服务鉴权失败 | 不重试，转 L2/L3 降级 |
| `40301` | 越权（请求了非开放的气象数据） | 拒绝并说明 |
| `40404` | 校区无坐标配置 / 该时段无预报 | 询问校区或说明超出预报范围 |
| `42900` | 外部 API 配额耗尽 | 走 L1/L2，不重试 |
| `50010` | 内部错误 | 降级 |
| `50410` | 上游超时 | 已内部降级；仍失败走 L3 |
| `50420` | 全部降级失败 | 使用 `message` 兜底话术 |
| `50300` | 熔断中 | 直接降级，不重试 |

---

## 9. 可观测性

| 项 | 说明 |
| --- | --- |
| 日志 | 结构化 JSON，必带 `trace_id`、`request_id`、`action`、`code`、`latency_ms`、`source`、`provider` |
| 指标 | `skill_calls_total{tool,action,code}`、`skill_latency_seconds`、`skill_degraded_total{level}`、`skill_circuit_state`、`skill_cache_hit_ratio`、`weather_upstream_quota_remaining` |
| 健康检查 | `/health` 返回自身与气象服务连通性；上游不健康时仍返回 200，仅标记 `upstream:"down"`，避免被误摘除 |
| 追踪 | 透传 `X-Trace-Id`，接入 OpenTelemetry |
| 配额告警 | 外部 API 日配额使用 > 80% 触发 P2 告警 |

---

## 10. 合规与隐私

| 项 | 要求 |
| --- | --- |
| 数据不入知识库 | 天气属实时数据，按 `docs/05 §1.1 D1` 不写入 PostgreSQL 知识库，也不写入用户画像 |
| 位置最小化 | 默认使用**校区级预设坐标**；仅用户显式授权时才接受精确坐标，且只在本轮会话内使用，不落盘 |
| 预警口径 | `alerts` 的类型/等级/标题/起止时间必须与气象台发布一致，禁止改写、禁止推测、禁止编造；无预警返回空数组 |
| 极端天气 | 必须追加"以学校官方通知为准"，不做避险承诺、不发布停课停业信息 |
| 健康场景 | 中暑/冻伤等只返回客观数值，就医与处置指引由出站护栏注入 |
| 数据来源标注 | 回答中必须体现 `provider` 与 `issued_at` |

---

## 11. 测试

| 类型 | 用例 |
| --- | --- |
| 契约测试 | 3 个 action ×（正常 / 边界 / 非法参数） |
| 规则测试 | `dining_advice` 映射表全覆盖（7 条分支 + 优先级叠加） |
| 归一化测试 | 校区名 → 坐标映射；`meal_periods` → 时段切片；日期越界 |
| 故障注入 | 上游超时（→ L1/L2）、上游 500（→ L1）、配额耗尽（→ 42900 → L1/L2）、Redis 宕机（→ L2/L3）、全失败（→ L4 兜底） |
| 合规测试 | 预警字段原样引用、无预警时不编造、不回传用户精确坐标 |
| 性能测试 | 50 QPS 持续 5 分钟，P95 ≤ 400ms，错误率 < 1% |

---

## 12. 目录结构（实现阶段）

```
agent/tools/weather-query/
├── README.md            ✅ 本文档
├── openapi.yaml         🔲 OpenAPI 3.1 契约
├── config.yaml          🔲 超时/重试/熔断/缓存/限流配置
├── Dockerfile           🔲
├── requirements.txt     🔲
├── src/
│   ├── main.py          🔲 FastAPI 应用与路由
│   ├── schemas.py       🔲 入出参 Pydantic 模型（与 openapi 对齐）
│   ├── service.py       🔲 查询编排与餐次切片
│   ├── advice.py        🔲 dining_advice 规则映射（纯函数）
│   ├── adapters/
│   │   ├── weather_api.py   🔲 外部气象服务适配器
│   │   ├── campus_geo.py    🔲 校区 → 坐标配置
│   │   ├── cache.py         🔲 Redis 缓存
│   │   └── climatology.py   🔲 季节气候基线（L3）
│   ├── resilience.py    🔲 超时/重试/熔断/降级链
│   └── errors.py        🔲 错误码
└── tests/               🔲 契约测试 + 规则测试 + 故障注入测试
```
