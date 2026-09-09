# Tool · `feedback-ticket` 投诉建议工单（写操作）

> ⚠️ **迁移声明**：原位于 `skills/feedback-ticket/`，经 skill↔tool 评审（见 `docs/00-项目结构.md §3`）判定为"逻辑确定（idempotency_key、分类枚举、字段校验）、可程序化调用"的写操作工具，已迁入 `agent/tools/`。写后仍由 LangGraph `interrupt` 人工确认。

| 项 | 值 |
| --- | --- |
> 编号 **T3**（见 `docs/05-技能清单与接口契约.md`）。

| 名称 | `feedback-ticket` |
| 版本 | `0.4.0` |
| 端口 | `8203`（工具段 8200+） |
| 类型 | **写**（唯一写操作工具） |
| 依赖 | Postgres（工单库）、工单通知通道 |
| SLA | P95 ≤ 800ms |
| 特殊约束 | **必须经主智能体 LangGraph `interrupt` 人工确认后才真正提交** |

---

## 1. 它做什么

把用户的投诉、建议、报修结构化为工单，生成预览 → 等待用户确认 → 提交 → 返回工单号与处理 SLA。

**不做**：承诺处理结果、赔付、处罚；不代替官方渠道做最终裁定。

---

## 2. 动作

| action | 说明 | 是否需确认 |
| --- | --- | --- |
| `create_ticket` | 创建工单（**先生成 `pending_confirm` 预览**） | ✅ |
| `confirm_ticket` | 用户确认后正式提交 | 由主智能体调用 |
| `cancel_ticket` | 取消 | ✅（用户点取消） |
| `query_ticket` | 按工单号查进度 | ❌ |

---

## 3. 请求示例

```json
{
  "request_id": "req_ULID",
  "skill": "feedback-ticket",
  "version": "0.4.0",
  "action": "create_ticket",
  "params": {
    "idempotency_key": "req_ULID",
    "category": "complaint",
    "sub_category": "service_attitude",
    "campus": "主校区",
    "canteen": "一食堂",
    "window": "2F-轻食窗口",
    "occurred_at": "2026-09-08T11:30:00+08:00",
    "description": "窗口工作人员态度生硬",
    "urgency": "normal",
    "contact": { "phone_masked": "138****5678", "preferred_channel": "phone" },
    "attachments": []
  },
  "context": { "user_id_hash": "u_xxx", "trace_id": "tr_xxx", "deadline_ms": 800 }
}
```

**`category` 枚举**：`complaint` | `suggestion` | `repair` | `food_safety` | `lost_and_found`
**`urgency`**：`low` | `normal` | `high`（命中"异物/变质/腹泻/食物中毒"自动置 `high` 并触发值班人告警）

> `lost_and_found`（失物招领）对应知识库 `03_营业与服务/食堂规章制度.md` 中的场景：创建工单的同时应把知识库中的**联系部门与联系方式**一并返回给用户；知识库未填写时走"暂未查到 + 建议到值班台询问"。

---

## 4. 响应示例（待确认）

```json
{
  "request_id": "req_ULID",
  "code": 0,
  "message": "ok",
  "data": {
    "ticket_id": "TK20260908000123",
    "status": "pending_confirm",
    "preview": "工单预览：类别=服务态度投诉 / 地点=主校区一食堂 2F 轻食窗口 / 时间=2026-09-08 11:30",
    "estimated_response_hours": 24
  },
  "meta": {
    "latency_ms": 120, "skill_version": "0.4.0",
    "source": "db", "degraded": false,
    "warnings": ["该工单需用户确认后提交"]
  }
}
```

主智能体收到 `status=pending_confirm` 后**必须**挂起图（`interrupt_before=["human_confirm"]`），把 `preview` 展示给用户等待确认。

---

## 5. 容错与合规

| 机制 | 说明 |
| --- | --- |
| **幂等** | `idempotency_key` 相同 → 直接返回首次创建结果，杜绝重复工单 |
| **不重试** | 写操作失败不自动重试；返回具体缺失项（如"缺少联系方式"）由主智能体追问用户 |
| 确认超时 | 10 分钟未确认 → 主智能体自动取消，工单置 `cancelled` |
| 联系方式 | 入参即脱敏（`138****5678`），库中加密存储，仅工单处理人可见 |
| 敏感升级 | 命中食安/异物/腹泻 → `urgency=high` + 同步值班人告警 + 返回就医与上报指引话术 |
| 降级 | 工单库不可用 → `50420` + "当前无法提交，请拨打后勤服务热线 xxx 或直接到食堂值班台登记" |
| 限流 | 单用户 5 单/天，防刷 |

---

## 6. 给 LLM 的工具描述

```yaml
name: feedback_ticket
description: >
  把用户的投诉、建议、报修结构化为工单。调用后会生成工单预览，
  必须先向用户展示预览并取得明确确认（"确认提交"），确认后才能正式提交；
  不要替用户擅自提交。需要先收集：类别、食堂/窗口、时间、具体描述。
  不要用于：查询（用 canteen_menu_query）、营养（用 nutrition_analyzer）。
when_to_use:
  - "我要投诉二楼窗口"
  - "建议增加素食窗口"
  - "二食堂空调坏了，报修"
  - "我水杯落在食堂了"
when_not_to_use:
  - 用户只是抱怨但没说要投诉 → 先共情，询问是否需要提交工单
  - 只是问"饭卡去哪补办" → 走 RAG（canteen_rules）
examples:
  - q: "我要投诉，今天中午二楼阿姨态度很差"
    params: { category: "complaint", sub_category: "service_attitude", canteen: "一食堂", window: "2F", description: "..." }
  - q: "我水杯丢在一食堂了"
    params: { category: "lost_and_found", canteen: "一食堂", description: "..." }
```

---

## 7. 错误码

| code | 含义 | 主智能体动作 |
| --- | --- | --- |
| `0` | 成功 | 展示预览 → 等待用户确认 |
| `40001` | 必填信息缺失 | 追问用户缺失字段 |
| `40301` | 越权/违规内容 | 拒绝并说明 |
| `42900` | 超出单用户日限额 | 告知明日再试或走热线 |
| `50010` | 内部错误 | 告知暂不可提交 + 热线 |
| `50420` | 工单库不可用 | 用兜底话术引导线下渠道 |
