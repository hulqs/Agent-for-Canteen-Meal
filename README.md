# 干饭宝 · 校园食堂智能助手（Agent for Canteen Meal）

> 一句话：面向高校师生的食堂场景对话式智能体。用 **LangGraph** 编排、**ReAct** 范式驱动推理与工具调用，用 **PostgreSQL + pgvector 构建 GraphRAG** 知识库（向量召回 + 全文召回 + 知识图谱增强）承载菜单与规章知识，把菜单/营养/人流/工单等能力封装成**独立可部署的 HTTP 技能**供主智能体自动调度，全链路配置**安全护栏**。

---

## 1. 它解决什么问题

| 角色 | 痛点 | 干饭宝的解法 |
| --- | --- | --- |
| 学生 | "今天二楼有什么？哪个窗口人少？15 块能吃啥？" | 一句话问出今日菜单、余量、供应时段、排队时长与预算内组合 |
| 特殊饮食人群 | 过敏、忌口、清真、减脂、宗教信仰 | 过敏原硬过滤 + 营养目标约束 + 民族餐饮尊重策略 |
| 食堂运营方 | 咨询重复、投诉散落、菜品数据沉睡 | 7×24 自动答疑、工单结构化沉淀、菜品数据反哺采购 |
| 学校管理者 | 食品安全舆情、数据合规 | 合规话术统一、风险提示强制注入、全流程审计留痕 |

---

## 2. 核心能力矩阵

| 编号 | 能力 | 主要实现方式 | 依赖能力 |
| --- | --- | --- | --- |
| C1 | 今日菜品 / 价格 / 窗口 / 余量 / 供应时段查询 | 工具调用（实时接口） | T1 `canteen-menu-query` |
| C2 | 个性化菜品推荐（口味、预算、忌口、营养目标、天气） | 工具调用 + 技能 + RAG | S1 `dish-recommender` → T1、T4 |
| C3 | 营养成分估算与膳食搭配建议 | 技能 + RAG（食物成分知识） | S2 `nutrition-analyzer` |
| C4 | 排队人流预测 / 错峰建议 | 工具调用 | T2 `crowd-forecast`（可选 T4 天气特征） |
| C5 | 食堂规章问答（营业时间、支付方式、失物招领、资助政策） | GraphRAG（PostgreSQL + pgvector） | — |
| C6 | 投诉建议、报修与失物招领工单 | 工具调用 + **人工确认中断** | T3 `feedback-ticket` |
| C7 | 食品安全与过敏原风险提示 | 安全护栏（强制注入） | — |
| C8 | 多轮追问、上下文记忆、用户偏好长期记忆 | LangGraph Checkpointer + Profile Store | — |
| C9 | 天气查询与就餐出行提示（实况/预报/官方预警） | 工具调用（外部气象服务） | T4 `weather-query` |

---

## 3. 技术选型

| 层次 | 选型 | 说明 |
| --- | --- | --- |
| 编排框架 | **LangGraph** | 有状态图、条件边、子图、持久化 Checkpoint、Human-in-the-loop 中断 |
| 推理范式 | **ReAct**（Thought → Action → Observation → …） | 以子图形式实现循环，带最大轮次与早停 |
| 向量数据库 | **PostgreSQL + pgvector**（GraphRAG） | 向量、全文、图谱同库同事务；HNSW 索引 + `tsvector` 中文全文 + 实体关系图；数据不出校 |
| 大模型 | 主模型（推理/工具选择）+ 轻量模型（护栏/改写/Embedding Rerank） | 双模型分层控成本 |
| 技能 / 工具形态 | 独立 HTTP 微服务（FastAPI）+ OpenAPI 契约 | 技能（8100+）与工具（8200+）均可单独部署、单独压测、单独灰度 |
| 缓存/熔断 | Redis + 本地 LRU | 四级降级：L0 实时 → L1 缓存 → L2 快照/基线 → L3 向量召回 → L4 兜底话术 |
| 可观测 | OpenTelemetry Trace + 结构化日志 + 指标看板 | 每次会话可回放完整 ReAct 轨迹 |
| 部署 | Docker Compose（PoC）→ K8s（生产） | 校内私有化，数据不出校 |

---

## 4. 四大硬性约束如何在本项目中落地

| 约束 | 落地位置 | 文档 |
| --- | --- | --- |
| ① 至少 1 个独立封装技能（HTTP 接口/工具节点/工作流，标准 JSON 入出参、可被主智能体自动调度、有容错） | **6 个能力**：`skills/` 下 2 个技能（S1/S2）+ `agent/tools/` 下 4 个工具（T1–T4），统一信封 + 错误码 + 熔断降级；主智能体通过 Tool Registry 自动发现与调度 | [技能封装规范](docs/04-技能封装规范.md) · [能力清单](docs/05-技能清单与接口契约.md) |
| ② 安全护栏（敏感内容过滤、风险提示、合规问答） | 入站闸门（注入检测/PII 脱敏/意图分级）+ 出站闸门（事实核验/合规改写/免责声明注入） | [安全护栏](docs/07-安全护栏.md) |
| ③ RAG 访问向量数据库（PostgreSQL + pgvector） | 7 个知识域（`kb` schema 分区表）、向量 + 全文 + 图谱混合检索、GraphRAG 社区摘要、增量 upsert、溯源引用 | [GraphRAG 与 PostgreSQL 向量存储](docs/06-GraphRAG知识库与PostgreSQL向量存储.md) · [数据库 Schema 与 SQL](docs/11-数据库Schema与SQL清单.md) |
| ④ LangGraph 架构 + ReAct 范式 | 主图（护栏→规划→检索→行动→观察→反思→出栏）+ ReAct 子图循环 | [总体架构](docs/02-总体架构.md) · [状态机与 ReAct](docs/03-LangGraph状态机与ReAct范式.md) |

---

## 5. 目录结构（详见 [docs/00-项目结构.md](docs/00-项目结构.md)）

```
Agent-for-Canteen-Meal/
├── README.md                  # 本文件
├── docs/                      # 项目文档（本仓库当前仅产出 .md）
├── skills/                    # ★ 技能：需模型推理/评估（8100+ 端口段）
│   ├── _template/             # 技能脚手架模板与规范
│   ├── dish-recommender/      # S1 个性化推荐（可调 T1/T4）
│   └── nutrition-analyzer/    # S2 营养分析
├── agent/                     # 主智能体（LangGraph，目录在文档中定义，代码后续实现）
│   └── tools/                 # ★ 工具：确定性执行、显式参数、结构化返回（8200+ 端口段）
│       ├── _template/         # 工具脚手架模板
│       ├── canteen-menu-query/  # T1 ★标杆：菜单/价格/余量/供应时段
│       ├── crowd-forecast/      # T2 人流预测（可选天气特征）
│       ├── feedback-ticket/     # T3 投诉工单（写操作，需人工确认）
│       └── weather-query/       # T4 天气查询（实况/预报/预警）
└── data/                      # 知识库原始语料（与「食堂干饭助手-知识库」目录一一对应）
```

> 调用方向：**主智能体 → 技能 / 工具**；**技能 → 工具**（允许，且只能是非强依赖的可选特征）；**禁止 skill→skill 与 tool→skill**（详见 `docs/00-项目结构.md §3.2`）。

---

## 6. 一次典型对话的执行链路

```
用户："明天中午一食堂有没有不辣的鸡胸肉，15 块以内？我对花生过敏。"

① 入站护栏 → PII 脱敏、注入检测、识别"过敏"= 高风险意图（标记 must_warn）
② 规划节点 → ReAct 第 1 轮：Thought「需要真实菜单数据」
              Action: tool_call T1 canteen-menu-query(date=+1, canteen=一食堂, meal_period=lunch,
                                                      filters={price_max:15, spice_level_max:0,
                                                               allergen_exclude:[花生]})
③ 工具执行 → 超时 800ms → 上游超时 → 重试 1 次 → 熔断未开 → 降级到 60s 缓存
              → 返回 data_version + degraded 标记 + 每道菜的 verified_date
④ ReAct 第 2 轮：Observation「命中 3 道菜，其中 2 道含辣标记」→ Thought「需核对辣度与过敏原交叉污染」
              Action: rag_retrieve(domain=food_safety, q="花生 交叉污染 加工线")
⑤ RAG → PostgreSQL(pgvector) 召回 5 段（仅 status=published）→ Rerank → 取 Top3，带 doc_id/chunk_id 溯源
⑥ 反思节点 → 校验「价格≤15 ✓ / 无花生 ✓ / 数据非实时 ✗ 需标注」
⑦ 出站护栏 → 强制注入："过敏信息仅供参考，请到窗口向工作人员确认配料与交叉污染情况"
⑧ 输出 → 菜品卡片 + 价格 + 窗口 + 供应时段 + 核验日期 + 数据来源时间戳 + 风险提示 + 引用来源
```

**再看一个多能力协同的例子：**

```
用户："今天中午下雨吗？一食堂吃点啥好，15 块以内。"

① ReAct 第 1 轮（并行，互不依赖）
     tool_call T4 weather_query.get_forecast(campus, date=今天, meal_periods=[lunch])
     tool_call T1 canteen_menu_query.query_dishes(canteen=一食堂, meal_period=lunch, price_max=15)
② observe → 天气：小雨、降水概率 0.7、walk_comfort=poor；菜单命中 5 道
③ ReAct 第 2 轮 → 需要"搭配"而非"列举"
     skill_call S1 dish_recommender(budget=15, context_features.weather="rain_light")
     → S1 内部强依赖 T1 取候选（命中上轮缓存），天气作为排序权重（雨天偏好就近/热食）
④ 出站护栏 → 附天气 provider + issued_at；命中 T1 缓存则附"以现场为准"；
              若有官方气象预警，追加"以学校官方通知为准"
```

> 注意：T4 不可用时，S1/T2 只是退化为无天气基线（`weather_used=false`），**不会**整体失败——这是"可选特征非强依赖"原则（见 `docs/05 §1.1 D5`）。

---

## 7. 文档索引

| 文档 | 内容 |
| --- | --- |
| [00-项目结构](docs/00-项目结构.md) | 完整目录树、每个目录职责、命名规范 |
| [01-需求与场景](docs/01-需求与场景.md) | 用户角色、用例清单、功能/非功能需求、明确不做的事 |
| [02-总体架构](docs/02-总体架构.md) | 分层架构、模块职责、请求时序、技术选型理由 |
| [03-LangGraph 状态机与 ReAct](docs/03-LangGraph状态机与ReAct范式.md) | State 定义、节点/边、ReAct 子图、中断与记忆 |
| [04-技能封装规范](docs/04-技能封装规范.md) | ★ 技能五要素、JSON 信封、错误码、容错、注册与自动调度 |
| [05-技能清单与接口契约](docs/05-技能清单与接口契约.md) | ★ 6 个能力（T1–T4 工具 / S1–S2 技能）的完整入出参、调用关系与 SLA |
| [06-GraphRAG 与 PostgreSQL 向量存储](docs/06-GraphRAG知识库与PostgreSQL向量存储.md) | ★ 写入链路、分块与向量化、存储设计、混合检索 + 图谱增强、质量与风险 |
| [07-安全护栏](docs/07-安全护栏.md) | ★ 四道闸门、敏感分类处置、注入防御、合规话术库 |
| [08-数据与接口设计](docs/08-数据与接口设计.md) | 数据模型、对外 API、流式、限流 |
| [09-评测与可观测性](docs/09-评测与可观测性.md) | 评测集、指标、追踪、告警 |
| [10-实施计划与部署](docs/10-实施计划与部署.md) | 五阶段里程碑、部署拓扑、容量估算 |
| [11-数据库 Schema 与 SQL 清单](docs/11-数据库Schema与SQL清单.md) | ★ 可执行建库 DDL、索引、写入/检索/巡检 SQL、权限 |

---

## 8. 状态

> 当前仓库**仅产出设计文档（.md）**，不包含实现代码。所有目录结构、接口契约、状态机定义均为后续编码阶段的实施依据。
