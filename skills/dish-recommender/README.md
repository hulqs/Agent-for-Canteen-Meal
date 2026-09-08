# Skill · `dish-recommender` 个性化菜品推荐

| 项 | 值 |
| --- | --- |
| 名称 | `dish-recommender` |
| 版本 | `0.2.0` |
| 端口 | `8102` |
| 类型 | 只读（组合优化） |
| 依赖 | 内部调用 `canteen-menu-query` 获取候选、Redis（偏好缓存） |
| SLA | P95 ≤ 700ms |

---

## 1. 它做什么

在给定约束（预算、忌口、过敏原、营养目标、口味）下，从当日真实候选中给出 **1 主菜 + 1 素菜 + 1 主食（+汤）** 的组合推荐，并给出可读的推荐理由。

**不做**：直接回答"今天有什么菜"（→ `canteen-menu-query`）；不做营养精算（→ `nutrition-analyzer`，本技能只做粗估用于排序）。

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
    "campus": "东校区", "canteen": "第一食堂",
    "date": "2026-09-08", "meal_period": "lunch",
    "budget": 15,
    "preferences": { "taste": ["清淡"], "avoid": ["香菜", "内脏"], "allergens": ["花生"] },
    "goals": { "type": "fat_loss", "calorie_max": 650 },
    "diversity": { "exclude_dish_ids": ["d_10087"], "reason": "最近已吃过" },
    "top_k": 3
  },
  "context": { "user_id_hash": "u_xxx", "trace_id": "tr_xxx", "deadline_ms": 800 }
}
```

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
          { "dish_id": "d_10087", "name": "香煎鸡胸饭", "price": 12.0, "role": "main" },
          { "dish_id": "d_10233", "name": "清炒时蔬", "price": 3.0, "role": "veg" }
        ],
        "total_price": 15.0,
        "nutrition_estimate": { "kcal": 610, "protein_g": 38, "fat_g": 14 },
        "reasons": ["蛋白质 38g 达单餐目标", "总价刚好 15 元", "已排除花生与香菜"],
        "score": 0.91
      }
    ],
    "constraints_applied": { "budget": 15, "allergen_exclude": ["花生"], "avoid": ["香菜", "内脏"] }
  },
  "meta": { "latency_ms": 240, "skill_version": "0.2.0", "source": "realtime", "degraded": false }
}
```

---

## 5. 容错设计

| 情况 | 处理 | `code` / 标记 |
| --- | --- | --- |
| 候选为空（预算过低/过滤过严） | 返回 `40404` + `data.suggestions`（如"最低可行预算 12 元""是否放宽辣度"） | `40404` |
| `canteen-menu-query` 超时/不可用 | 降级为 ChromaDB 历史菜单近似推荐 | `degraded=true`，`source=vector_recall` |
| 求解超时 300ms | 返回束搜索当前最优解（保证有结果） | `degraded=true`，`warnings` 说明 |
| 营养目标不可满足 | 返回最接近方案 + 说明差距（"最低 680 kcal，略超你的 650 目标"） | `0` + `warnings` |
| 用户画像缺失 | 使用默认偏好（均衡、中位预算） | `profile=default` |

硬约束（**任何降级路径都不允许放宽**）：过敏原过滤、忌口过滤。

---

## 6. 给 LLM 的工具描述

```yaml
name: dish_recommender
description: >
  在预算、忌口、过敏原、营养目标、口味等约束下，从当日真实菜单中
  给出"主菜+素菜+主食"的组合推荐，并附推荐理由和总价估算。
  当用户说"吃啥好""推荐一个""换换口味""15 块怎么搭配"时使用。
  若返回 40404，应依据 data.suggestions 向用户提出放宽建议。
  不要用于：单纯问有什么菜（canteen_menu_query）、要精确营养数据（nutrition_analyzer）。
when_to_use:
  - "15 块能怎么搭配？"
  - "我在减脂，推荐个午餐"
  - "吃点不一样的"
when_not_to_use:
  - "今天中午有什么菜" → canteen_menu_query
  - "这个组合多少卡" → nutrition_analyzer
examples:
  - q: "15 块吃啥好，我清淡为主"
    params: { budget: 15, preferences: { taste: ["清淡"] }, top_k: 3 }
```

---

## 7. 错误码

| code | 含义 | 主智能体动作 |
| --- | --- | --- |
| `0` | 成功 | 展示组合 + 理由 |
| `40001` | 参数不合法 | 修正后重试 |
| `40404` | 无可行组合 | 依据 `suggestions` 建议放宽条件 |
| `50410` | 上游菜单技能超时 | 已走历史降级，提示"参考历史数据" |
| `50420` | 上游完全不可用 | 用兜底话术 |
| `50300` | 熔断中 | 改走 RAG 或直接告知稍后再试 |
