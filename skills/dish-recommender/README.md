# Skill · `dish-recommender` 个性化菜品推荐

| 项 | 值 |
| --- | --- |
> 编号 **S1**（见 `docs/05-技能清单与接口契约.md`）。

| 名称 | `dish-recommender` |
| 版本 | `0.2.0` |
| 端口 | `8101`（技能段 8100+） |
| 类型 | 只读（组合优化） |
| 依赖 | **强依赖 T1 `canteen-menu-query`**（skill→tool HTTP 调用获取候选）、Redis（偏好缓存）；**可选** T4 `weather-query` 天气特征、S2 营养粗估 |
| SLA | P95 ≤ 700ms |

---

## 1. 它做什么

在给定约束（预算、忌口、过敏原、营养目标、口味、天气）下，从当日真实候选中给出 **1 主菜 + 1 素菜 + 1 主食（+汤）** 的组合推荐，并给出可读的推荐理由。

**不做**：直接回答"今天有什么菜"（→ T1 `canteen-menu-query`）；不做营养精算（→ S2 `nutrition-analyzer`，本技能只做粗估用于排序）；不查天气本身（→ T4 `weather-query`）。

---

## 2. 动作

| action | 说明 |
| --- | --- |
| `recommend_combo` | 推荐组合（Top-K） |
| `explain` | 解释某个已推荐组合的理由 |

---

## 3. 请求示例

```json
{
  "request_id": "req_ULID",
  "skill": "dish-recommender",
  "version": "0.2.0",
  "action": "recommend_combo",
  "params": {
    "campus": "主校区", "canteen": "一食堂",
    "date": "2026-09-08", "meal_period": "lunch",
    "budget": 15,
    "budget_tier": "10_15",
    "preferences": { "taste": ["清淡"], "avoid": ["香菜", "内脏"], "allergens": ["花生"] },
    "goals": { "type": "fat_loss", "calorie_max": 650 },
    "context_features": { "weather": "rain_light", "walk_comfort": "poor" },
    "diversity": { "exclude_dish_ids": ["d_10087"], "reason": "最近已吃过" },
    "top_k": 3
  },
  "context": { "user_id_hash": "u_xxx", "trace_id": "tr_xxx", "deadline_ms": 800 }
}
```

| 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `budget` | number | ❌ | 显式预算上限（元） |
| `budget_tier` | enum | ❌ | `le_10` / `10_15` / `gt_15`，对齐知识库 `05_推荐素材`「按预算推荐」三档；与 `budget` 同时传时取更严格者 |
| `context_features.weather` | string | ❌ | 天气码（来自 T4）；仅作排序权重，**不影响**过敏与忌口硬约束 |
| `context_features.walk_comfort` | enum | ❌ | `good`/`fair`/`poor`；`poor` 时偏好就近窗口与热食 |

---

## 4. 响应示例

```json
{
  "request_id": "req_ULID",
  "code": 0,
  "message": "ok",
  "data": {
    "combos": [
      {
        "rank": 1,
        "items": [
          { "dish_id": "d_10087", "name": "香煎鸡胸饭", "price": 12.0, "role": "main", "window": "2F-轻食窗口" },
          { "dish_id": "d_10233", "name": "清炒时蔬", "price": 3.0, "role": "veg", "window": "2F-轻食窗口" }
        ],
        "total_price": 15.0,
        "nutrition_estimate": { "kcal": 610, "protein_g": 38, "fat_g": 14 },
        "reasons": ["蛋白质 38g 达单餐目标", "总价刚好 15 元", "已排除花生与香菜"],
        "score": 0.91
      }
    ],
    "constraints_applied": { "budget": 15, "allergen_exclude": ["花生"], "avoid": ["香菜", "内脏"] }
  },
  "meta": { "latency_ms": 240, "skill_version": "0.2.0", "source": "realtime", "degraded": false,
            "weather_used": true }
}
```

---

## 5. 容错设计

| 情况 | 处理 | `code` / 标记 |
| --- | --- | --- |
| 候选为空（预算过低/过滤过严） | 返回 `40404` + `data.suggestions`（如"最低可行预算 12 元""可看 10–15 元档""是否放宽辣度"） | `40404` |
| T1 `canteen-menu-query` 超时/降级 | 继承 T1 的降级结果；若 T1 走 `vector_recall`，本技能一并置 `degraded=true` + 提示"参考历史数据" | `degraded=true`，`source=vector_recall` |
| T1 完全不可用（`50420`/`50300`） | 返回 `50420` + 兜底话术（**不重试**，由主智能体决定是否改走 RAG） | `50420` |
| T4 天气不可用 | 退化为无天气排序，继续返回组合 | `weather_used=false` + `warnings:["weather_feature_missing"]`，**不置 degraded** |
| 求解超时 300ms | 返回束搜索当前最优解（保证有结果） | `degraded=true`，`warnings` 说明 |
| 营养目标不可满足 | 返回最接近方案 + 说明差距（"最低 680 kcal，略超你的 650 目标"） | `0` + `warnings` |
| 用户画像缺失 | 使用默认偏好（均衡、中位预算） | `profile=default` |

> 推荐素材（特色菜卡、按预算推荐）来自 PostgreSQL `canteen_dish_features` 域，仅取 `status=published` 且带 `verified_date` 的条目。

硬约束（**任何降级路径都不允许放宽**）：过敏原过滤、忌口过滤。

---

## 6. 给 LLM 的工具描述

```yaml
name: dish_recommender
description: >
  在预算、忌口、过敏原、营养目标、口味、天气等约束下，从当日真实菜单中
  给出"主菜+素菜+主食"的组合推荐，并附推荐理由和总价估算。
  当用户说"吃啥好""推荐一个""换换口味""15 块怎么搭配""下雨天吃点啥"时使用。
  若返回 40404，应依据 data.suggestions 向用户提出放宽建议。
  需要天气特征时，先由主智能体调用 weather_query 再通过 context_features 传入，
  本技能不会自己去查天气；天气只影响排序，不影响过敏与忌口硬约束。
  不要用于：单纯问有什么菜（canteen_menu_query）、要精确营养数据（nutrition_analyzer）。
when_to_use:
  - "15 块能怎么搭配？"
  - "我在减脂，推荐个午餐"
  - "吃点不一样的"
  - "下雨天吃点啥好？"
when_not_to_use:
  - "今天中午有什么菜" → canteen_menu_query
  - "这个组合多少卡" → nutrition_analyzer
  - "今天下雨吗" → weather_query
examples:
  - q: "15 块吃啥好，我清淡为主"
    params: { budget: 15, preferences: { taste: ["清淡"] }, top_k: 3 }
  - q: "十块钱能吃什么"
    params: { budget_tier: "le_10", top_k: 3 }
  - q: "下雨天吃点啥好"
    params: { top_k: 3, context_features: { weather: "rain_light", walk_comfort: "poor" } }
```

---

## 7. 错误码

| code | 含义 | 主智能体动作 |
| --- | --- | --- |
| `0` | 成功 | 展示组合 + 理由 |
| `40001` | 参数不合法 | 修正后重试 |
| `40404` | 无可行组合 | 依据 `suggestions` 建议放宽条件（预算档/辣度/标签） |
| `50410` | 上游菜单工具（T1）超时 | 已走历史降级，提示"参考历史数据" |
| `50420` | 上游完全不可用 | 用兜底话术 |
| `50300` | 熔断中 | 改走 RAG 或直接告知稍后再试 |
