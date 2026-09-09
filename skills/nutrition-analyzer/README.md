# Skill · `nutrition-analyzer` 营养分析

| 项 | 值 |
| --- | --- |
> 编号 **S2**（见 `docs/05-技能清单与接口契约.md`）。

| 名称 | `nutrition-analyzer` |
| 版本 | `0.3.0` |
| 端口 | `8102`（技能段 8100+） |
| 类型 | 只读（估算） |
| 依赖 | 本地食物成分表、ChromaDB `nutrition_knowledge`、估算模型 |
| SLA | P95 ≤ 600ms |

---

## 1. 它做什么

估算菜品或组合的**能量与三大营养素**（kcal / 蛋白质 / 脂肪 / 碳水 / 钠），对照参考摄入量给出**参考性**评价。

**不做**：医疗诊断、疾病饮食处方、个体化营养方案（这类请求一律降级为通用原则 + 建议咨询校医院/注册营养师）。

---

## 2. 动作

| action | 说明 |
| --- | --- |
| `analyze` | 分析给定菜品/组合的营养构成 |
| `compare` | 对比两个组合（如"哪个更适合减脂"） |
| `suggest_intake` | 按用户画像给出单餐参考摄入区间（**仅参考值**） |

---

## 3. 请求示例

```json
{
  "request_id": "req_ULID",
  "skill": "nutrition-analyzer",
  "version": "0.3.0",
  "action": "analyze",
  "params": {
    "items": [
      { "name": "香煎鸡胸饭", "dish_id": "d_10087", "portion": "1份" },
      { "name": "清炒时蔬", "dish_id": "d_10233", "portion": "1份" }
    ],
    "user_profile": { "gender": "male", "age": 20, "weight_kg": 70, "activity_level": "moderate" },
    "goal": "fat_loss"
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
    "totals": { "kcal": 610, "protein_g": 38.2, "fat_g": 14.5, "carb_g": 78.0, "sodium_mg": 1120 },
    "reference": { "recommended_kcal_per_meal": 700, "protein_target_g": 35 },
    "evaluation": [
      { "dim": "energy", "level": "ok", "text": "能量 610 kcal，低于单餐参考 700 kcal，符合减脂节奏" },
      { "dim": "sodium", "level": "high", "text": "钠含量偏高，建议搭配清淡汤品或减少酱汁" }
    ],
    "confidence": "medium",
    "disclaimer": "REQUIRED_NUTRITION_DISCLAIMER"
  },
  "meta": {
    "latency_ms": 210, "skill_version": "0.3.0", "source": "成分表+估算模型",
    "degraded": false,
    "warnings": ["2 项依据同类菜品中位值估算，存在 ±20% 误差"]
  }
}
```

> `disclaimer` 是**契约级必填字段**：主智能体出站护栏会据此强制展开为"以上为估算值，不构成医疗或营养处方建议"。缺失该字段时护栏按高风险处理。

---

## 5. 容错设计

| 情况 | 处理 | `code` / 标记 |
| --- | --- | --- |
| 菜品不在成分表 | 用同类菜品中位值估算 | `confidence=low` + `meta.warnings` |
| 成分表服务不可用 | 降级为 ChromaDB `nutrition_knowledge` 语义召回的近似值 | `degraded=true`，`source=vector_recall` |
| 完全无法估算 | 返回 `40404` + "暂无法估算该菜品，建议咨询食堂公示的营养信息" | `40404` |
| 用户画像缺失 | 用同年龄段默认值，标注 `profile=default` | `confidence=medium` |
| 求解超时 300ms | 返回当前已完成的部分结果（缺项置 null） | `degraded=true` |

**合规硬约束**
- 检测到疾病关键词（糖尿病、肾病、痛风、肝病等）→ 不输出具体数值处方，仅返回通用膳食原则 + `disclaimer` + 建议就医
- 涉及"减肥药""断食""催吐"等 → 返回 `40301` 并附健康提示话术

---

## 6. 给 LLM 的工具描述

```yaml
name: nutrition_analyzer
description: >
  估算菜品或一餐组合的能量与三大营养素（kcal/蛋白质/脂肪/碳水/钠），
  并对照参考摄入量给出参考性评价。当用户问"多少卡""蛋白质够不够""这个健康吗"时使用。
  返回值含 confidence，low 时必须告知用户是估算值；必须保留 disclaimer 提示。
  不要用于：查菜单（canteen_menu_query）、排队（crowd_forecast）。
when_to_use:
  - "这顿大概多少卡？"
  - "帮我看看蛋白质够不够"
  - "减脂吃这个行吗"
when_not_to_use:
  - "今天有什么菜" → canteen_menu_query
  - "我糖尿病该怎么吃" → 不给处方，引导就医（本技能仅返回通用原则）
examples:
  - q: "鸡胸饭加青菜多少卡"
    params: { items: [{ name: "香煎鸡胸饭" }, { name: "清炒时蔬" }], goal: "maintain" }
```

---

## 7. 错误码

| code | 含义 | 主智能体动作 |
| --- | --- | --- |
| `0` | 成功 | 正常使用（保留 disclaimer） |
| `40001` | 参数不合法（如 items 为空） | 修正后重试 |
| `40301` | 涉及高风险健康请求 | 不给处方，输出健康提示 |
| `40404` | 无法估算 | 告知用户并建议其他渠道 |
| `50010` | 内部错误 | 降级或告知暂不可用 |
| `50410` | 成分表服务超时 | 走向量召回降级 |
