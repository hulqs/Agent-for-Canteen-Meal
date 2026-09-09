# Tool Template · 工具脚手架模板

> 新增工具时：复制本目录 → 改名为 `<tool-name>` → 逐项填写标 `【待填】` 的位置 → 走完文末检查清单 → 在 `agent/tools/registry.py` 注册。

---

## 【待填】`<tool-name>`

| 项 | 值 |
| --- | --- |
| 名称 | 【待填】小写连字符，如 `campus-bus-query` |
| 版本 | `0.1.0` |
| 端口 | 【待填】8205+（已占用：8201 菜单 / 8202 人流 / 8203 工单 / 8204 天气） |
| 类型 | 读 / 写 |
| 依赖 | 【待填】上游服务、缓存、数据库 |
| SLA | P95 ≤ 500ms，可用性 ≥ 99.5% |
| 是否需人工确认 | 读=否；写=**是**（LangGraph `interrupt`） |
| 是否会被技能调用 | 【待填】是/否；若是，必须声明"可选特征非强依赖"（调用方不得因本工具失败而整体失败） |

---

## 1. 它做什么

【待填】一句话说明工具能力，并明确**不做**什么。

> 与"技能"的区别：本工具**不依赖模型推理**，输入参数 schema 固定，返回结构化 JSON。调用方（主智能体或其他技能）按需显式传入参数即可获得确定结果。

---

## 2. 契约

```
POST /v1/{action}
GET  /health
GET  /openapi.json
GET  /metrics
```

| action | 说明 |
| --- | --- |
| 【待填】 | 【待填】 |

---

## 3. 请求 / 响应信封

> 与技能信封**完全一致**：请求 `{request_id, skill→tool, version, action, params, context}`，响应 `{request_id, code, message, data, meta}`。统一 5 位错误码、四级降级。详见 `docs/04-技能封装规范.md`（该规范同时适用于工具）。

> ⚠️ 实现阶段注意：请求字段名沿用 `skill` 以兼容 Registry / Invoker 复用代码；若团队倾向用 `tool` 字段名，应在 OpenAPI 与 Schema 同步修改。

---

## 4. 容错与可观测性

要求同技能：超时、重试（仅读）、熔断、四级降级、兜底话术、幂等（写）、限流、缓存、`/health` `/metrics` 暴露。完整约束见 `docs/04-技能封装规范.md`。

---

## 5. 注册与被调用

工具由 `agent/tools/registry.py` 注册为主智能体可直接调用的 Tool（LangChain StructuredTool 形式），被调方式有两种：

1. **主智能体直接调用**：ReAct 循环中模型自主选择工具名 + 参数。
2. **被技能调用**：技能内部通过 HTTP 调用本工具（本项目内不允许 skill-to-skill 直接调用，但允许 skill-to-tool）。

---

## 6. 上线检查清单

- [ ] 业务边界单一，与已有技能/工具无重叠
- [ ] `openapi.yaml` + `README.md` + `examples` 齐全
- [ ] 入参 JSON Schema 完整（enum、范围、必填）
- [ ] 出参遵循统一信封
- [ ] 超时 / 重试 / 熔断 / 四级降级（L0–L4）/ 兜底话术已实现
- [ ] 写操作具备幂等键 + 人工确认中断
- [ ] `/health` `/metrics` `/openapi.json` 暴露
- [ ] 日志打印 `trace_id`，指标接入 Prometheus
- [ ] 已在 `agent/tools/registry.py` 注册
- [ ] 已在 `docs/05-技能清单与接口契约.md` 的**总表、附 A（场景矩阵）、附 B（文档索引）**三处同步登记
- [ ] 已在 `docs/00-项目结构.md` 端口表续号；枚举取值与 `docs/08 §1.4` 字典一致
- [ ] 实时类数据已确认不写入 ChromaDB