# Tool · `crowd-forecast` 窗口人流与排队预测

> ⚠️ **迁移声明**：原位于 `skills/crowd-forecast/`，经 skill↔tool 评审（见 `docs/00-项目结构.md §3`）判定为"显式参数、结构化返回、可程序化调用"的工具能力，已迁入 `agent/tools/`。

| 项 | 值 |
| --- | --- |
> 编号 **T2**（见 `docs/05-技能清单与接口契约.md`）。

| 名称 | `crowd-forecast` |
| 版本 | `0.1.0` |
| 端口 | `8202`（工具段 8200+） |
| 类型 | 只读（预测） |
| 依赖 | 脱敏聚合流水、历史基线模型、Redis；**可选** T4 `weather-query`（天气特征，非强依赖） |
| SLA | P95 ≤ 600ms |

---

## 1. 它做什么

预测各窗口未来 30/60 分钟的排队时长与人流等级，给出错峰建议。

**不做**：输出任何个人消费记录或明细（合规红线，只输出聚合值）；不做精确承诺；不直接推荐菜品（→ `dish-recommender`）。

> 天气只是**可选预测特征**：`with_weather=true` 时内部调用 T4 `weather-query` 取降雨概率与气温；T4 不可用 → 置 `weather_used=false` 走无天气基线，**不置 `degraded=true`**（`docs/05 §1.1 D5`）。

---

## 2. 动作

| action | 说明 |
| --- | --- |
| `forecast` | 预测指定食堂各窗口排队时长 |
| `best_time` | 给出建议到店时间 |

---

## 3. 请求示例

```json
{
  "request_id": "req_ULID",
  "skill": "crowd-forecast",
  "version": "0.1.0",
  "action": "forecast",
  "params": {
    "campus": "主校区",
    "canteen": "一食堂",
    "datetime": "2026-09-08T11:50:00+08:00",
    "horizon_minutes": 30,
    "with_weather": true
  },
  "context": { "user_id_hash": "u_xxx", "trace_id": "tr_xxx", "deadline_ms": 800 }
}
```

| 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `campus` / `canteen` | string | ❌ | 与 T1 共用枚举字典（见 `docs/08 §1.4`） |
| `datetime` | string(date-time) | ❌ | 缺省 = 当前时间 |
| `horizon_minutes` | int | ❌ | 30 / 60，默认 30 |
| `with_weather` | bool | ❌ | 默认 false；置 true 时把 T4 天气作为可选特征 |

---

## 4. 响应示例

```json
{
  "request_id": "req_ULID",
  "code": 0,
  "message": "ok",
  "data": {
    "windows": [
      { "window": "2F-轻食窗口", "queue_minutes": 2, "level": "low" },
      { "window": "1F-大锅菜", "queue_minutes": 12, "level": "high" }
    ],
    "best_time_to_go": "12:20 后预计回落",
    "advice": "现在 2 楼轻食窗口基本不用排队"
  },
  "meta": {
    "latency_ms": 180, "skill_version": "0.1.0", "source": "model_v2",
    "degraded": false, "confidence": 0.78, "weather_used": true,
    "warnings": []
  }
}
```

> `meta.confidence` < 0.6 时，主智能体必须在回答中加"预测仅供参考"。

---

## 5. 容错设计

| 情况 | 处理 | 标记 |
| --- | --- | --- |
| 实时流水缺失/延迟 | 用"同星期几 + 同时间段"历史基线 | `degraded=true`，`source=baseline` |
| 模型服务不可用 | 返回最近一次成功预测结果（若有）并标注时间 | `degraded=true`，`source=last_known` |
| 新窗口/无历史 | 返回 `40404` + "该窗口暂无足够历史数据" | `40404` |
| 置信度 < 0.6 | 正常返回但提示仅供参考 | `confidence` 低 |
| T4 天气不可用 | 退化为无天气基线，继续返回预测 | `weather_used=false` + `warnings:["weather_feature_missing"]`，**不置 degraded** |
| 完全失败 | `50420` + "暂时无法预测，建议避开 11:40–12:20 高峰" | `50420` |

**隐私硬约束**：工具层对输出做聚合校验——任何包含个人标识、单笔消费、身份信息的字段在出口前被剥离；请求参数不接受个人级查询。

---

## 6. 给 LLM 的工具描述

```yaml
name: crowd_forecast
description: >
  预测各食堂窗口未来 30/60 分钟的排队时长与人流等级，给出错峰建议。
  当用户问"哪个窗口人少""现在去要不要排队""什么时候去不用等""下雨天人会不会少点"时使用。
  若 meta.confidence 低于 0.6 或 degraded 为 true，回答时必须加"预测仅供参考"。
  本工具会自行决定是否使用天气特征，不需要你先单独查天气；
  若用户只问天气，请用 weather_query。
  不要用于：查菜单（canteen_menu_query）、推荐吃什么（dish_recommender）。
when_to_use:
  - "哪个窗口人少？"
  - "现在去要不要排队？"
  - "什么时候去人少"
  - "下雨天人会不会少点？"
when_not_to_use:
  - "今天有什么菜" → canteen_menu_query
  - "今天下雨吗" → weather_query
examples:
  - q: "现在一食堂人多吗"
    params: { canteen: "一食堂", horizon_minutes: 30 }
  - q: "下雨天人会不会少点"
    params: { canteen: "一食堂", horizon_minutes: 30, with_weather: true }
```

---

## 7. 错误码

| code | 含义 | 主智能体动作 |
| --- | --- | --- |
| `0` | 成功 | 展示预测 + 建议（低置信时加提示） |
| `40001` | 参数不合法 | 修正后重试 |
| `40404` | 该窗口无历史数据 | 告知并给通用错峰建议 |
| `50010` | 内部错误 | 降级 |
| `50420` | 预测服务不可用 | 通用错峰建议兜底 |
