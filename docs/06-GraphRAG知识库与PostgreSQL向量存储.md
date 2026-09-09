# 06 · GraphRAG 知识库与 PostgreSQL 向量存储 ★

> **本次改造**：知识库底座由 ChromaDB 迁移为 **PostgreSQL 原生方案 + pgvector 插件**，检索范式由「纯向量 RAG」升级为 **GraphRAG**（向量召回 + 全文召回 + 知识图谱关系增强 + 社区摘要）。
>
> 知识库唯一事实来源仍是 **「食堂干饭助手-知识库」**（本地目录），经 `data/` 镜像后由 indexer 摄入 PostgreSQL。本文件定义"知识库目录 → 解析 → 切分 → 向量化 → 入库 → 图构建 → 检索 → 生成"的完整数据流，以及每一步的输入输出契约。
>
> 完整可执行的建库脚本、索引 DDL、写入/检索/巡检 SQL 见 **[11-数据库Schema与SQL清单](11-数据库Schema与SQL清单.md)**。

---

## 0. 改造摘要（与旧方案的对照）

| 维度 | 旧方案（ChromaDB） | 新方案（PostgreSQL + pgvector + GraphRAG） |
| --- | --- | --- |
| 存储引擎 | ChromaDB 独立服务 + 持久化卷 | PostgreSQL 16+（已有实例）+ `vector` 扩展，与业务库同实例、不同 schema（`kb`） |
| 组织单位 | 7 个 Collection | 7 个 **domain**（`kb.domain` 枚举），落为 `kb.chunk` / `kb.chunk_embedding` 的 **LIST 分区** |
| 元数据过滤 | Collection metadata（弱类型 KV） | 强类型列 + `CHECK` 约束 + `jsonb`（低频字段），可建索引、可外键、可事务 |
| 向量索引 | HNSW（引擎内置，参数有限） | **HNSW 为主**（每分区一个 partial index）+ IVFFlat 为大表备选，参数可调、可 `REINDEX CONCURRENTLY` |
| 全文检索 | 需应用层自建 BM25 | `tsvector` + GIN（`zhparser` 中文分词）+ `pg_trgm` 模糊匹配，与向量检索**同库同事务** |
| 图谱能力 | 无 | `kb.entity` / `kb.relation` / `kb.chunk_entity` / `kb.community`，递归 CTE 做 1–2 跳关系增强召回 |
| 一致性 | 文本与向量分别写入，无事务保证 | 文档级事务 + **outbox 队列** + `embed_state` 状态机 + `is_current` 原子切换，检索永不看到"有文本无向量"的中间态 |
| 溯源 | `doc_id` / `chunk_id` | 同上，**且图谱每条边必须有 `relation_evidence` 证据 chunk**，无证据的边不得用于回答 |
| 运维 | 独立备份、独立监控 | 复用 PG 的 WAL / PITR / 主从 / `pg_stat_*` / VACUUM，运维面收敛为一套 |

**保留不变的铁律**（改造不放松任何一条）：

1. 事实性内容（价格、时间、政策、位置、余量）必须来自检索召回或能力返回；无召回走"知识不足"分支。
2. **实时量不入库**：拥挤度、排队时长、饭卡余额、天气只由 T2/T4 工具提供。
3. **未核验不作答**：缺 `verified_date` 或状态非 `published` 一律按"暂未查到"处理。
4. 检索默认硬过滤 `status='published'`，任何场景不得关闭。

---

## 1. 为什么选 PostgreSQL 原生 + pgvector + GraphRAG

| 决策 | 理由 |
| --- | --- |
| **用 PG 承载向量，而非独立向量库** | ① 项目本就有 Postgres（工单/审计/画像），少一套服务＝少一套备份、监控、故障域；② 元数据过滤是本场景的**主路径**（食堂/楼层/餐次/日期/状态），关系型的强类型列 + 索引远优于 KV metadata；③ 向量与业务表可 **JOIN**（如 chunk ↔ 菜品实体 ↔ 工单），跨库做不到；④ 事务保证"文本、向量、图谱"三者一致，这是旧方案最大的隐患。 |
| **pgvector 而非 pg_embedding 等** | 事实标准、活跃维护（0.8.x 系列持续迭代）、HNSW/IVFFlat 双索引、`halfvec` 半精度、**0.8+ 的 iterative index scan** 专门解决"带过滤条件时召回不足"——这正是本项目的核心风险点。 |
| **HNSW 为主索引** | 本项目 chunk 量级 5k–50k（远期 < 30 万），菜单类**每日增量 upsert**。HNSW 免训练、增量插入即生效、召回率高；IVFFlat 需要代表性数据训练聚类中心，增量写入会造成聚类漂移、必须周期重建，不适合日更场景。 |
| **升级为 GraphRAG** | 食堂场景的问题大量是**关系型**的："一食堂二楼有哪些窗口""这道菜含不含花生""有清真选择吗""哪家便宜"。纯向量召回只能命中"字面相似的段落"，而答案往往分散在多篇文档里（菜品表 + 过敏原说明 + 楼层布局）。图谱把"食堂—楼层—窗口—菜品—过敏原—政策"显式连起来，一跳就能补齐纯向量漏掉的证据。 |
| **社区摘要（Leiden）** | 支撑"全局型"提问（"三个食堂各有什么特色""便宜又管饱的有哪些"）。这类问题没有单一 chunk 能回答，需要主题级摘要做 map-reduce。 |
| **不引入 Neo4j/AGE** | 图规模小（实体 < 5 万、边 < 20 万），1–2 跳查询用递归 CTE 足够，`EXPLAIN` 可控；引入图数据库会再多一个故障域，且图与向量又要跨库对齐——违背本次"收敛底座"的初衷。 |

**版本要求**

| 组件 | 版本 | 原因 |
| --- | --- | --- |
| PostgreSQL | **16+**（17 亦可） | 分区表触发器、`bit_count()`、并行索引构建、`REINDEX CONCURRENTLY` 成熟度 |
| pgvector | **≥ 0.8.0**（建议 0.8.6） | iterative index scan（过滤召回）、`halfvec`、HNSW 扫描性能改进 |
| zhparser | 任意 | 中文分词；无法安装时降级为 `pg_bigm` 或 `simple` 配置 + `pg_trgm`（见 §7.3 R-09） |
| pg_trgm / btree_gin / pgcrypto | 随 PG 发行 | 别名模糊匹配、复合 GIN、`gen_random_uuid()` |

---

## 2. 数据流全景

### 2.1 写入链路（离线，Ingestion）

```
「食堂干饭助手-知识库」目录
  │ ① 快照同步（rsync + sha256）
  ▼
data/ 镜像 + manifest.jsonl ──────────────────► kb.ingest_run（登记一次运行）
  │ ② 解析（Markdown/PDF/Word/Excel/JSON）
  ▼  RawDoc{source_path, front_matter, blocks[]}
  │ ③ 清洗（去页眉页脚、表格还原为行文本）
  ▼  CleanDoc
  │ ④ 状态门控（status/verified_date）── 不合规 ──► kb.reject_log（拒绝入库，可追溯）
  ▼
  │ ⑤ 切分（按 domain 策略，见 §3.2）
  ▼  Chunk[]{seq, content, breadcrumb, char_len, token_len}
  │ ⑥ 归一化 + 富化（辣度/餐次/别名/楼层/价格/预算档 → 强类型列 + attrs jsonb）
  ▼
  │ ⑦ 去重（content_sha256 精确 + SimHash 近似）
  ▼
  │ ⑧ 【事务 A】写 kb.source_document + kb.chunk（is_current=false, embed_state=pending）
  │              同事务写 kb.embedding_job（outbox）
  ▼
  │ ⑨ 向量化 worker：FOR UPDATE SKIP LOCKED 领取 → batch=64 embed → L2 归一化
  │              【事务 B】写 kb.chunk_embedding + chunk.embed_state='ready'
  ▼
  │ ⑩ 图抽取 worker：实体/关系/提及 upsert（kb.entity / relation / relation_evidence / chunk_entity）
  ▼
  │ ⑪ 社区检测 + 摘要（Leiden，离线）→ kb.community / community_member（+ summary 向量）
  ▼
  │ ⑫ 【事务 C】kb.publish_document()：全部 chunk ready 才把 is_current 切到新版本（原子切换）
  ▼
  │ ⑬ ANALYZE + 抽样回测（Recall@5）→ 写回 kb.ingest_run，不达标则告警并可一键回滚
  ▼
                              检索可见
```

### 2.2 检索链路（在线，Query）

```
用户问题（已脱敏）
  │ ① 查询改写：指代消解 / 时间归一 / 实体抽取
  ▼
  │ ② 实体链接：kb.entity_alias（精确 → trgm 模糊 → 实体向量兜底）→ seed_entities[]
  ▼
  │ ③ 域路由：意图 → domain[]（分区裁剪，避免全库广播）
  ▼
  │ ④ 问题向量化（与入库同一 model_key，同一归一化方式）→ qvec
  ▼
  ├─ ⑤a Dense：kb.chunk_embedding HNSW，`embedding <=> qvec` Top-30
  ├─ ⑤b Sparse：kb.chunk.tsv GIN，`ts_rank_cd` Top-30（解决菜名/窗口号精确匹配）
  └─ ⑤c Graph：seed_entities 递归 1–2 跳 → chunk_entity → 关联 chunk Top-20
  ▼
  │ ⑥ RRF 融合（k=60，权重 dense 1.0 / sparse 0.8 / graph 0.6）
  ▼
  │ ⑦ Rerank（Cross-Encoder）→ Top-8
  ▼
  │ ⑧ MMR 去冗余（λ=0.7）→ Top-5 ；阈值过滤（cos_sim < 0.35 丢弃）
  ▼
  │ ⑨ 权威源去重（同 canteen+category 命中 primary 与 secondary，保留 primary）
  ▼
  │ ⑩ 组装上下文（chunk_ref / 面包屑 / verified_date / source_path / 图谱路径说明）
  ▼
  │ ⑪ 写 kb.retrieval_log（供偏差检测与回归）
  ▼
主智能体生成 → 出站护栏核验引用 → 用户
```

> **全局型问题**（"三个食堂各有什么特色"）走另一条支路：⑤ 换成对 `kb.community.summary_embedding` 检索，取 Top-3 社区摘要做 map-reduce，再回落到成员实体的 chunk 取证。见 §5.4。

---

## 3. 写入链路：从文档到向量的每一步衔接

### 3.1 阶段契约总表（输入 → 输出 → 传递形式 → 幂等键）

| # | 阶段 | 输入 | 输出 | 环节间传递形式 | 幂等键 | 失败处置 |
| --- | --- | --- | --- | --- | --- | --- |
| ① | 快照同步 | 知识库目录（文件系统） | `data/` 镜像 + `manifest.jsonl` | 文件 + JSONL 每行 `{source_path, sha256, mtime, size, bytes}` | `source_path + sha256` | 整次运行终止，不写库 |
| ② | 解析 | `data/` 单文件 | `RawDoc` | 进程内 Python dataclass：`RawDoc(source_path, front_matter: dict, blocks: list[Block])` | `source_path` | 记 `reject_log(stage='parse')`，跳过该文件 |
| ③ | 清洗 | `RawDoc` | `CleanDoc` | 同上，`blocks` 中表格已还原为「菜品｜价格｜辣度｜供应时段｜特色描述」行文本 | `source_path` | 同上 |
| ④ | 状态门控 | `CleanDoc.front_matter` | 通过 / 拒绝 | 布尔 + 原因码 | `source_path` | `reject_log(stage='gate', reason_code)`；`draft` 直接跳过，`expired` 走软删除 |
| ⑤ | 切分 | `CleanDoc` | `Chunk[]` | `list[Chunk(seq, content, breadcrumb, char_len, token_len)]` | `doc_id + seq` | 拒绝整篇（避免半篇入库） |
| ⑥ | 归一化+富化 | `Chunk[]` + front_matter | `EnrichedChunk[]` | 追加强类型字段（`canteen/floor/meal_period/spice_level/price/budget_tier/allergens[]`）+ `attrs jsonb` | `doc_id + seq` | 单条降级：无法识别的字段置 `NULL` + 写 `reject_log(stage='normalize', severity='warn')`；**禁止猜默认值** |
| ⑦ | 去重 | `EnrichedChunk[]` | 去重后 `Chunk[]` | 计算 `content_sha256`、`simhash` | `content_sha256` | 重复直接丢弃并计数 |
| ⑧ | 落库（事务 A） | `Chunk[]` | `kb.source_document` 1 行 + `kb.chunk` N 行（`is_current=false`, `embed_state='pending'`）+ `kb.embedding_job` N 行 | **数据库行**（此后不再走进程内传递） | `source_path+doc_version` / `(doc_id,seq,domain)` | 整事务回滚，`ingest_run.status='failed'` |
| ⑨ | 向量化（事务 B） | `kb.embedding_job` 中 pending 行 → `kb.chunk.content` + `breadcrumb` | `kb.chunk_embedding` N 行 + `chunk.embed_state='ready'` | **数据库行**；embedding 为 `vector(1024)` | `(chunk_id, domain, model_key)` | `attempts+1`，指数退避重试 3 次；仍失败 → `embed_state='failed'` + 告警，该 chunk **不进入检索** |
| ⑩ | 图抽取 | 已 ready 的 chunk | `kb.entity` / `kb.relation` / `kb.relation_evidence` / `kb.chunk_entity` | 数据库行 | `(entity_type,name_norm)` / `(head,type,tail,valid_from)` / `(chunk_id,domain,entity_id,char_start)` | 单 chunk 失败只影响图增强召回，不影响向量召回（**图是增强，不是前置依赖**） |
| ⑪ | 社区检测 | `kb.entity` + `kb.relation` 全图 | `kb.community` / `kb.community_member` | 数据库行 + `summary_embedding` | `graph_run_id + level + 成员集合哈希` | 保留上一版 `is_current` 社区，不影响在线 |
| ⑫ | 发布（事务 C） | `doc_id` | `is_current` 原子切换 | 数据库行 | `doc_id` | 不切换，旧版本继续服务（**读不到半成品**） |
| ⑬ | 校验 | 黄金集 | `ingest_run.recall_at5` | 数据库行 + 告警 | `run_id` | Recall@5 < 85% → 告警 + 可执行 `kb.rollback_document()` |

**关键工程决策**：从阶段 ⑧ 起，数据的唯一载体是**数据库行**，不再有"内存中的中间态"。阶段 ⑨–⑫ 全部由独立 worker 通过读库驱动，因此 indexer 进程崩溃、重启、并发扩容都不会导致数据丢失或重复——这是"整条数据流逻辑完整"的根本保证。

### 3.2 分块策略与块大小设定

**通用参数**

| 参数 | 值 | 说明 |
| --- | --- | --- |
| 目标块长 | **200–500 汉字** | bge-m3 在中文短段落上语义最稳；过短丢上下文，过长稀释主题 |
| 硬上限 | 600 汉字（`CHECK char_len <= 1200` 字符，含面包屑与标点余量） | 超限强制二次切分，DB 层兜底拦截 |
| 重叠 | **15%**（30–75 汉字） | 只在"按长度切"时启用；结构化切分不重叠 |
| 切分边界 | 句号 / 分号 / 换行 / 表格行边界优先，**禁止切在数字与单位之间**（避免"12 / 元"被切开） | 价格类事实的正确性依赖此规则 |
| 面包屑前置 | 每块首行注入 `主校区 > 一食堂 > 二楼 > 营业时间` | 同时进入向量与 `tsvector`，显著提升"二楼有什么"这类问法的命中 |

**分域策略**

| domain | 切分单位 | 块大小 | 是否重叠 | 理由 |
| --- | --- | --- | --- | --- |
| `canteen_profile` | 食堂 × 楼层 | ≤ 400 字 | 否 | 问"二楼有什么"要整段命中 |
| `canteen_menu_docs` | 食堂 + 窗口 + 日期（表格转行文本后整块） | ≤ 600 字 | 否 | 保证"一天一窗口"上下文完整；单菜品另建实体入图，不再单独切块 |
| `canteen_rules` | 条款 / 章节（非固定长度） | 200–400 字 | 否 | 政策条款语义完整，切断即失真 |
| `canteen_faq` | 一问一答 | 不切 | 否 | 检索粒度 = 答案粒度 |
| `canteen_dish_features` | 一张特色菜卡 / 一个预算档整表 | ≤ 300 字 | 否 | 便于按 `budget_tier` 精确过滤 |
| `nutrition_knowledge` | 一条食物条目 | ≤ 300 字 | 否 | 便于按 `food_name` 精确过滤 |
| `food_safety` | 风险主题 | 300–500 字 | **是（15%）** | 长篇说明按长度切，需重叠保上下文 |

> **切分与图谱的分工**：菜品这类"实体级信息"不切成小块（会碎片化），而是①随窗口块整体入向量，②同时抽成 `kb.entity(entity_type='dish')` 并连边到窗口/食堂/过敏原。查"某道菜含不含花生"走图谱，查"这个窗口都有什么"走向量——两条路各司其职。

### 3.3 向量化（Embedding）规格

| 项 | 规格 | 说明 |
| --- | --- | --- |
| 模型 | `bge-m3`（dense 分支），**1024 维**，校内本地部署 | 数据不出校；中文短文本表现优于 384 维小模型，对菜名类短查询更稳 |
| `model_key` | `bge-m3@v1.5` | **模型 + 版本**写入每一行向量；换模型必须换 key，禁止混用（见 §7.2 R-05） |
| 归一化 | 写入前 **L2 归一化**，`is_normalized=true` | 归一化后余弦距离与内积等价，`<=>` 结果可直接用 `1 - distance` 解释为余弦相似度 |
| 距离算子 | `vector_cosine_ops` / `<=>` | 与模型训练目标一致；即便向量已归一化也保留 cosine 算子（可读性优先，性能差异 < 5%） |
| 批大小 | 64 | 显存与吞吐平衡；worker 并发 2–4 |
| 输入文本 | `breadcrumb || '\n' || content` | 与检索期的查询构造保持**同一拼接方式**，否则语义空间偏移 |
| 重试 | 3 次指数退避（1s/4s/16s） | 失败落 `embed_state='failed'`，不阻塞其他 chunk |
| 幂等 | `ON CONFLICT (chunk_id, domain, model_key) DO UPDATE` | 重跑安全 |

**内容未变则不重算向量**：`kb.chunk` upsert 时比较 `content_sha256`，一致则保留原 `embed_state='ready'` 与原向量。这既省算力，也避免"同一文本因模型非确定性产生微小漂移"导致的检索结果抖动。

```sql
-- 阶段 ⑧ 的 chunk upsert 核心片段（完整版见 docs/11 §5.2）
INSERT INTO kb.chunk (chunk_id, domain, doc_id, seq, chunk_ref, breadcrumb,
                      content, content_sha256, simhash, char_len, token_len,
                      status, authority, verified_date, campus, canteen, floor,
                      business_date, meal_period, spice_level, price, budget_tier,
                      allergens, attrs, embed_state, is_current, ingest_run_id)
VALUES (...)
ON CONFLICT (doc_id, seq, domain) DO UPDATE SET
  content        = EXCLUDED.content,
  content_sha256 = EXCLUDED.content_sha256,
  ...
  -- 内容未变 → 沿用原状态与原向量；内容变了 → 打回 pending 重新向量化
  embed_state = CASE WHEN kb.chunk.content_sha256 = EXCLUDED.content_sha256
                     THEN kb.chunk.embed_state ELSE 'pending' END,
  updated_at  = now();
```

### 3.4 事务边界与 outbox：为什么不能"边切边写向量"

朴素做法是"切一块 → 调 embedding → 写一行"，它有三个致命问题：① embedding 服务抖动会让文档半篇入库；② 长事务持有锁，阻塞日更；③ 无法并发扩容 worker。

本方案的解法是 **outbox（发件箱）模式 + 状态机**：

```
事务 A（快、纯本地）           异步 worker（慢、可重试、可并发）        事务 C（原子发布）
─────────────────────         ───────────────────────────────         ──────────────────
source_document  ──┐          FOR UPDATE SKIP LOCKED 领取 job          全部 chunk ready?
chunk(pending)   ──┼─ COMMIT  → embed(batch=64)                        ├ 是 → is_current=true
embedding_job    ──┘          → chunk_embedding + embed_state=ready    └ 否 → 不切换，告警
```

| 保证 | 机制 |
| --- | --- |
| 不丢 | `embedding_job` 与 `chunk` 在**同一事务**写入，只要 chunk 落库，任务必然存在 |
| 不重 | `(chunk_id, domain, model_key)` 主键 + `ON CONFLICT DO UPDATE` |
| 不串 | `FOR UPDATE SKIP LOCKED` 保证多 worker 不领同一批 |
| 不脏 | 检索视图只认 `embed_state='ready' AND is_current`，中间态天然不可见 |
| 可回滚 | 旧版本行保留（`is_current=false`），`kb.rollback_document()` 一句切回 |

**`embed_state` 状态机**

```
pending ──worker 领取──► running ──成功──► ready ──内容变更──► pending
   ▲                        │                │
   └───retry(<3)────────────┘                └──文档下线──► （随 is_current=false 退出检索）
                            └──retry 耗尽──► failed（告警，人工介入）
```

### 3.5 图构建：实体、关系、社区

**实体抽取（规则优先，模型兜底）**

| 实体类型 | 抽取方式 | 来源 |
| --- | --- | --- |
| `campus` / `canteen` / `window` / `floor` | **规则**（枚举字典 + 别名表精确匹配） | 知识库 01、02 结构化字段 |
| `dish` | **规则**（菜品表逐行）+ 文本中的模型 NER 补充 | 知识库 02、05 |
| `allergen` / `nutrient` | 规则（受控词表） | 知识库 nutrition / food_safety |
| `policy` / `faq` / `hazard` | 章节标题 + 模型抽取 | 知识库 03、04、food_safety |
| `budget_tier` / `meal_period` / `tag` | 规则（枚举） | 归一化产物 |

> 规则优先的原因：食堂域的核心实体是**封闭集合**（3 个食堂、2 层楼、有限窗口），规则抽取准确率接近 100%，且可解释、可回归。模型只用于开放描述里的补充抽取，且 `confidence < 0.7` 的实体不参与图增强召回。

**关系类型（`kb.relation_type_dict`）**

| relation_type | head → tail | 来源 | 用于回答 |
| --- | --- | --- | --- |
| `LOCATED_IN` | canteen → campus | 知识库 01 | 食堂在哪 |
| `ON_FLOOR` | window → floor | 知识库 01 | 二楼有什么 |
| `HAS_WINDOW` | canteen → window | 知识库 01/02 | 某食堂有哪些窗口 |
| `SERVES` | window → dish | 知识库 02 | 这个窗口卖什么 / 这道菜在哪买 |
| `CONTAINS_ALLERGEN` | dish → allergen | 知识库 02/food_safety | **过敏场景（高风险）** |
| `MAY_CONTAIN` | dish → allergen | food_safety（交叉污染） | 过敏场景的风险提示 |
| `HAS_NUTRIENT` | dish → nutrient | nutrition | 营养问答（S2） |
| `IN_BUDGET_TIER` | dish → budget_tier | 知识库 05 | 15 块能吃啥 |
| `AVAILABLE_AT` | dish → meal_period | 知识库 02 | 早上有没有 |
| `SUBSTITUTE_OF` | dish → dish | 知识库 05 + 模型 | 换一个类似的 |
| `GOVERNED_BY` | canteen/window → policy | 知识库 03 | 支付方式 / 失物招领 |
| `ANSWERS` | faq → policy/dish | 知识库 04 | FAQ 定位 |
| `WARNS_ABOUT` | hazard → dish/allergen | food_safety | 食安提示注入 |

**每条边必须有证据**：`kb.relation_evidence(relation_id, chunk_id, domain, quote)` 记录该关系来自哪个 chunk 的哪句话。检索时图谱路径会连带证据 chunk 一起返回，出站护栏据此校验引用真实性。**无证据的边视为脏数据，`kb.v_graph_edge` 视图直接排除**——这条约束是"图谱不引入新幻觉"的关键。

**社区检测与摘要**

| 项 | 设定 |
| --- | --- |
| 算法 | Leiden（`igraph`/`graspologic`），离线执行，写入 `kb.community` |
| 层级 | `level=0` 细粒度（如"一食堂二楼轻食窗口群"）、`level=1` 中粒度（如"一食堂全貌"）、`level=2` 全局（如"全校清真供应"） |
| 边权 | `relation.weight × confidence`；`CONTAINS_ALLERGEN` 等高风险边权重固定为 1.0 |
| 摘要生成 | 对社区内实体 + 其证据 chunk 做 LLM 摘要，**摘要内不得出现证据 chunk 中不存在的数字**（生成后用规则校验数字集合是否为子集，不通过则重试 1 次，仍失败则只保留实体名列表） |
| 摘要向量 | 同 `model_key`，写入 `community.summary_embedding` |
| 更新频率 | 每日全量重算（图规模小，秒级–分钟级）；`is_current` 切换，旧版本保留 1 天 |

---

## 4. 存储设计

### 4.1 Schema 划分与表清单

所有知识库对象放在 **`kb`** schema，与业务库（`app`：工单、会话、审计、画像）物理同实例、逻辑隔离。检索账号 `kb_reader` 只授予 `kb` 下**视图**的 SELECT，不授予基表——防止应用绕过 `status` 硬过滤。

| 分组 | 表 | 作用 | 行数量级 |
| --- | --- | --- | --- |
| **文档层** | `kb.source_document` | 文档级元数据、状态、核验、版本、权威源 | 10²–10³ |
| **文本层** | `kb.chunk`（按 domain LIST 分区） | 文本块 + 强类型过滤列 + `tsvector` + `embed_state` | 10³–10⁵ |
| **向量层** | `kb.chunk_embedding`（按 domain LIST 分区） | 向量 + `model_key` + `is_searchable` 冗余标志 | 10³–10⁵（× 模型数） |
| **图层** | `kb.entity` | 实体节点（含实体描述向量） | 10³–10⁴ |
| | `kb.entity_alias` | 别名 → 实体（落地 docs/08 §1.4 别名表） | 10³ |
| | `kb.relation` | 关系边（带时效 `valid_from/valid_to`） | 10⁴–10⁵ |
| | `kb.relation_evidence` | 边的证据 chunk（**边必须有证据**） | ≥ relation |
| | `kb.chunk_entity` | chunk ↔ 实体提及（M:N，带位置与显著度） | 10⁴–10⁶ |
| | `kb.community` / `kb.community_member` | 社区及摘要（GraphRAG global search） | 10²–10³ |
| **字典层** | `kb.entity_type_dict` / `kb.relation_type_dict` | 类型字典（可增量扩展，不用 ENUM） | 10¹ |
| **运维层** | `kb.ingest_run` / `kb.graph_run` | 入库/图构建运行记录（版本、耗时、Recall） | 10²–10³ |
| | `kb.embedding_job` | 向量化 outbox 队列 | 波动 |
| | `kb.reject_log` | 拒绝入库明细（脏数据台账） | 10²–10⁴ |
| | `kb.alias_miss` | 别名未命中（驱动运营补录） | 10²–10³ |
| | `kb.retrieval_log` | 检索日志（偏差检测、回归、看板） | 10⁵+（30 天分区滚动） |
| | `kb.eval_golden` | 检索黄金集（q → 期望 chunk_ref） | 10² |

**为什么 `domain` 用 ENUM 而 `entity_type` / `relation_type` 用字典表**：`domain` 是分区键，新增一个域＝新建分区＝架构变更，必须走评审，ENUM 的"改起来麻烦"正是我们想要的约束；而实体/关系类型会随语料演进频繁增补，用字典表 + 外键才能在不停机的情况下扩展。

### 4.2 核心 DDL（节选，完整版见 docs/11）

```sql
-- ============ 扩展与枚举 ============
CREATE SCHEMA IF NOT EXISTS kb;
CREATE EXTENSION IF NOT EXISTS vector;      -- pgvector >= 0.8.0
CREATE EXTENSION IF NOT EXISTS pg_trgm;     -- 别名模糊匹配
CREATE EXTENSION IF NOT EXISTS btree_gin;
CREATE EXTENSION IF NOT EXISTS pgcrypto;    -- gen_random_uuid()

CREATE TYPE kb.domain AS ENUM (
  'canteen_profile', 'canteen_menu_docs', 'canteen_rules', 'canteen_faq',
  'canteen_dish_features', 'nutrition_knowledge', 'food_safety');
CREATE TYPE kb.corpus_status AS ENUM ('draft', 'published', 'expired');
CREATE TYPE kb.authority     AS ENUM ('primary', 'secondary', 'index');
CREATE TYPE kb.embed_state   AS ENUM ('pending', 'running', 'ready', 'failed');

-- 中文分词配置（zhparser 不可用时见 §7.3 R-09 降级方案）
CREATE TEXT SEARCH CONFIGURATION kb.zh (PARSER = zhparser);
ALTER  TEXT SEARCH CONFIGURATION kb.zh ADD MAPPING FOR n,v,a,i,e,l,j,t WITH simple;

-- ============ 文档层 ============
CREATE TABLE kb.source_document (
  doc_id          text             PRIMARY KEY,             -- doc_<sha1(source_path)[0:10]>
  domain          kb.domain        NOT NULL,
  source_path     text             NOT NULL,                -- 知识库相对路径
  source_uri      text             NOT NULL,                -- internal://kb/<source_path>
  title           text             NOT NULL,
  doc_version     int              NOT NULL DEFAULT 1,
  content_sha256  char(64)         NOT NULL,
  campus          text,
  canteen         text,
  category        text,                                     -- business_hours / payment / ...
  authority       kb.authority     NOT NULL DEFAULT 'primary',
  status          kb.corpus_status NOT NULL,
  verified_date   date,
  effective_date  date,
  expire_date     date,
  reviewer        text,
  reviewed_at     date,
  acl             text[]           NOT NULL DEFAULT '{student,staff,public}',
  risk_level      text             NOT NULL DEFAULT 'normal',
  kb_snapshot_at  timestamptz      NOT NULL,
  ingest_run_id   uuid             NOT NULL REFERENCES kb.ingest_run(run_id),
  is_current      boolean          NOT NULL DEFAULT false,
  deleted         boolean          NOT NULL DEFAULT false,
  created_at      timestamptz      NOT NULL DEFAULT now(),
  updated_at      timestamptz      NOT NULL DEFAULT now(),
  CONSTRAINT uq_doc_path_ver UNIQUE (source_path, doc_version),
  -- 把文档铁律下沉为数据库约束：published 必须有核验日期，脏数据在写入时就被拒绝
  CONSTRAINT ck_doc_verified CHECK (status <> 'published' OR verified_date IS NOT NULL),
  CONSTRAINT ck_doc_dates    CHECK (expire_date IS NULL OR effective_date IS NULL
                                    OR expire_date >= effective_date)
);
-- 同一 source_path 只允许一个 current 版本
CREATE UNIQUE INDEX uq_doc_current ON kb.source_document (source_path)
  WHERE is_current AND NOT deleted;

-- ============ 文本层（按 domain 分区）============
CREATE TABLE kb.chunk (
  chunk_id        uuid             NOT NULL DEFAULT gen_random_uuid(),
  domain          kb.domain        NOT NULL,                -- 分区键
  doc_id          text             NOT NULL,
  seq             int              NOT NULL,
  chunk_ref       text             NOT NULL,                -- 'doc_0031::chunk_007'，回答里的引用锚点
  breadcrumb      text             NOT NULL,                -- 主校区 > 一食堂 > 二楼 > 营业时间
  content         text             NOT NULL,
  content_sha256  char(64)         NOT NULL,
  simhash         bigint,                                   -- 64 位，近似去重
  char_len        int              NOT NULL,
  token_len       int              NOT NULL,
  -- 高频过滤字段提升为强类型列（可建索引、可 CHECK；不塞 jsonb）
  status          kb.corpus_status NOT NULL,
  authority       kb.authority     NOT NULL DEFAULT 'primary',
  verified_date   date,
  effective_date  date,
  expire_date     date,
  campus          text,
  canteen         text,
  window_name     text,
  floor           smallint,
  business_date   date,                                     -- 菜单类：本块描述哪一天
  meal_period     text,
  spice_level     smallint,
  price           numeric(6,2),
  budget_tier     text,
  dish_tags       text[]           NOT NULL DEFAULT '{}',
  allergens       text[]           NOT NULL DEFAULT '{}',
  hazard_type     text,
  attrs           jsonb            NOT NULL DEFAULT '{}'::jsonb,   -- 域特有低频字段
  -- 全文检索列：面包屑 + 正文一起分词，"二楼"这类词才能被 BM25 命中
  tsv             tsvector GENERATED ALWAYS AS
                    (to_tsvector('kb.zh'::regconfig, breadcrumb || ' ' || content)) STORED,
  embed_state     kb.embed_state   NOT NULL DEFAULT 'pending',
  is_current      boolean          NOT NULL DEFAULT false,
  deleted         boolean          NOT NULL DEFAULT false,
  ingest_run_id   uuid             NOT NULL,
  created_at      timestamptz      NOT NULL DEFAULT now(),
  updated_at      timestamptz      NOT NULL DEFAULT now(),
  PRIMARY KEY (chunk_id, domain),                           -- 分区表主键必须含分区键
  CONSTRAINT uq_chunk_doc_seq UNIQUE (doc_id, seq, domain),
  FOREIGN KEY (doc_id) REFERENCES kb.source_document(doc_id) ON DELETE CASCADE,
  -- 归一化规则的数据库兜底：越界值写不进来
  CONSTRAINT ck_chunk_floor CHECK (floor       IS NULL OR floor       BETWEEN 1 AND 2),
  CONSTRAINT ck_chunk_spice CHECK (spice_level IS NULL OR spice_level BETWEEN 0 AND 3),
  CONSTRAINT ck_chunk_price CHECK (price       IS NULL OR price > 0),
  CONSTRAINT ck_chunk_meal  CHECK (meal_period IS NULL OR
                                   meal_period IN ('breakfast','lunch','dinner')),
  CONSTRAINT ck_chunk_tier  CHECK (budget_tier IS NULL OR
                                   budget_tier IN ('le_10','10_15','gt_15')),
  CONSTRAINT ck_chunk_len   CHECK (char_len BETWEEN 1 AND 1200)
) PARTITION BY LIST (domain);

CREATE TABLE kb.chunk_p_menu    PARTITION OF kb.chunk FOR VALUES IN ('canteen_menu_docs');
CREATE TABLE kb.chunk_p_rules   PARTITION OF kb.chunk FOR VALUES IN ('canteen_rules');
CREATE TABLE kb.chunk_p_profile PARTITION OF kb.chunk FOR VALUES IN ('canteen_profile');
CREATE TABLE kb.chunk_p_faq     PARTITION OF kb.chunk FOR VALUES IN ('canteen_faq');
CREATE TABLE kb.chunk_p_feature PARTITION OF kb.chunk FOR VALUES IN ('canteen_dish_features');
CREATE TABLE kb.chunk_p_nutri   PARTITION OF kb.chunk FOR VALUES IN ('nutrition_knowledge');
CREATE TABLE kb.chunk_p_safety  PARTITION OF kb.chunk FOR VALUES IN ('food_safety');

-- ============ 向量层（按 domain 分区）============
CREATE TABLE kb.chunk_embedding (
  chunk_id      uuid         NOT NULL,
  domain        kb.domain    NOT NULL,                      -- 分区键
  model_key     text         NOT NULL,                      -- 'bge-m3@v1.5'
  dim           smallint     NOT NULL,
  embedding     vector(1024) NOT NULL,
  is_normalized boolean      NOT NULL DEFAULT true,
  -- 冗余可检索标志：由触发器从 chunk 同步而来，使 HNSW 能建 partial index（见 §4.4）
  is_searchable boolean      NOT NULL DEFAULT false,
  embedded_at   timestamptz  NOT NULL DEFAULT now(),
  ingest_run_id uuid         NOT NULL,
  PRIMARY KEY (chunk_id, domain, model_key),
  FOREIGN KEY (chunk_id, domain) REFERENCES kb.chunk(chunk_id, domain) ON DELETE CASCADE,
  CONSTRAINT ck_emb_dim CHECK (dim = 1024 AND vector_dims(embedding) = 1024)
) PARTITION BY LIST (domain);
-- 7 个同名分区，略（见 docs/11 §3.3）
```

**图层核心 DDL（节选）**

```sql
CREATE TABLE kb.entity (
  entity_id      uuid  PRIMARY KEY DEFAULT gen_random_uuid(),
  entity_type    text  NOT NULL REFERENCES kb.entity_type_dict(type_code),
  canonical_name text  NOT NULL,
  name_norm      text  NOT NULL,                    -- 去空格/全半角/小写后的归一名
  description    text,                              -- 实体摘要（GraphRAG）
  desc_embedding vector(1024),                      -- 用于"实体链接兜底"与相似实体推荐
  attrs          jsonb NOT NULL DEFAULT '{}'::jsonb,
  degree         int   NOT NULL DEFAULT 0,           -- 物化度数，用于超级节点剪枝
  status         kb.corpus_status NOT NULL DEFAULT 'published',
  confidence     numeric(4,3) NOT NULL DEFAULT 1.000,
  deleted        boolean NOT NULL DEFAULT false,
  created_at     timestamptz NOT NULL DEFAULT now(),
  updated_at     timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT uq_entity UNIQUE (entity_type, name_norm)
);

CREATE TABLE kb.relation (
  relation_id   uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  head_id       uuid NOT NULL REFERENCES kb.entity(entity_id) ON DELETE CASCADE,
  relation_type text NOT NULL REFERENCES kb.relation_type_dict(type_code),
  tail_id       uuid NOT NULL REFERENCES kb.entity(entity_id) ON DELETE CASCADE,
  weight        numeric(5,4) NOT NULL DEFAULT 1.0000,
  confidence    numeric(4,3) NOT NULL DEFAULT 1.000,
  valid_from    date NOT NULL DEFAULT DATE '1900-01-01',    -- 关系时效：菜品换季、政策修订
  valid_to      date,
  attrs         jsonb NOT NULL DEFAULT '{}'::jsonb,
  status        kb.corpus_status NOT NULL DEFAULT 'published',
  deleted       boolean NOT NULL DEFAULT false,
  created_at    timestamptz NOT NULL DEFAULT now(),
  updated_at    timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT uq_relation   UNIQUE (head_id, relation_type, tail_id, valid_from),
  CONSTRAINT ck_rel_noself CHECK (head_id <> tail_id),
  CONSTRAINT ck_rel_valid  CHECK (valid_to IS NULL OR valid_to >= valid_from)
);

CREATE TABLE kb.relation_evidence (           -- 边必须有证据，否则不得用于回答
  relation_id uuid      NOT NULL REFERENCES kb.relation(relation_id) ON DELETE CASCADE,
  chunk_id    uuid      NOT NULL,
  domain      kb.domain NOT NULL,
  quote       text      NOT NULL,             -- 原句片段，出站护栏据此校验
  extractor   text      NOT NULL,             -- rule / ner / llm
  confidence  numeric(4,3) NOT NULL DEFAULT 1.000,
  PRIMARY KEY (relation_id, chunk_id, domain),
  FOREIGN KEY (chunk_id, domain) REFERENCES kb.chunk(chunk_id, domain) ON DELETE CASCADE
);

CREATE TABLE kb.chunk_entity (                -- chunk ↔ 实体提及
  chunk_id   uuid      NOT NULL,
  domain     kb.domain NOT NULL,
  entity_id  uuid      NOT NULL REFERENCES kb.entity(entity_id) ON DELETE CASCADE,
  mention    text      NOT NULL,
  char_start int       NOT NULL,
  char_end   int       NOT NULL,
  salience   numeric(4,3) NOT NULL DEFAULT 0.500,   -- 该实体在此块中的重要度
  extractor  text      NOT NULL,
  confidence numeric(4,3) NOT NULL DEFAULT 1.000,
  PRIMARY KEY (chunk_id, domain, entity_id, char_start),
  FOREIGN KEY (chunk_id, domain) REFERENCES kb.chunk(chunk_id, domain) ON DELETE CASCADE
);
```

### 4.3 检索视图（唯一对外出口）

```sql
-- 应用只允许查这个视图，status 硬过滤下沉到数据库，无法被应用层"忘记"
CREATE VIEW kb.v_searchable_chunk AS
SELECT c.chunk_id, c.domain, c.doc_id, c.chunk_ref, c.breadcrumb, c.content,
       c.authority, c.verified_date, c.campus, c.canteen, c.window_name, c.floor,
       c.business_date, c.meal_period, c.spice_level, c.price, c.budget_tier,
       c.dish_tags, c.allergens, c.hazard_type, c.attrs, c.tsv,
       d.title, d.source_path, d.category, d.reviewer, d.doc_version
FROM   kb.chunk c
JOIN   kb.source_document d USING (doc_id)
WHERE  c.is_current AND NOT c.deleted
  AND  c.status = 'published'
  AND  c.embed_state = 'ready'                    -- 无向量的块不可见（数据流完整性）
  AND  c.verified_date IS NOT NULL                -- 未核验不作答
  AND  (c.expire_date IS NULL OR c.expire_date >= CURRENT_DATE)
  AND  d.is_current AND NOT d.deleted;

-- 无向图视图：图展开时不必关心边的方向
CREATE VIEW kb.v_graph_edge AS
SELECT r.relation_id, r.head_id AS src, r.tail_id AS dst, r.relation_type,
       r.weight, r.confidence, false AS reversed
FROM   kb.relation r
WHERE  r.status = 'published' AND NOT r.deleted
  AND  (r.valid_to IS NULL OR r.valid_to >= CURRENT_DATE)
  AND  EXISTS (SELECT 1 FROM kb.relation_evidence e WHERE e.relation_id = r.relation_id)
UNION ALL
SELECT r.relation_id, r.tail_id, r.head_id, r.relation_type,
       r.weight * 0.9, r.confidence, true
FROM   kb.relation r
WHERE  r.status = 'published' AND NOT r.deleted
  AND  (r.valid_to IS NULL OR r.valid_to >= CURRENT_DATE)
  AND  EXISTS (SELECT 1 FROM kb.relation_evidence e WHERE e.relation_id = r.relation_id);
```

> 反向边权重乘 0.9：正向语义（"窗口 → 供应 → 菜品"）比反向（"菜品 → 被供应于 → 窗口"）稍强，融合排序时体现出来。

### 4.4 索引设计与 HNSW / IVFFlat 选型

**向量索引（每个分区一个 partial HNSW）**

```sql
-- 只索引"可检索"的向量：索引更小、召回更准、重建更快
CREATE INDEX idx_emb_menu_hnsw ON kb.chunk_embedding_p_menu
  USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 200)
  WHERE is_searchable AND model_key = 'bge-m3@v1.5';
-- 其余 6 个分区同构（menu 数据量最大，其它可用 ef_construction = 128）

-- 查询期参数（每个会话/事务内设置）
SET LOCAL hnsw.ef_search        = 100;              -- 召回率 vs 延迟
SET LOCAL hnsw.iterative_scan   = 'relaxed_order';  -- pgvector 0.8+：过滤条件严时继续迭代扫描
SET LOCAL hnsw.max_scan_tuples  = 20000;            -- 迭代扫描上限，防止长尾查询打爆
```

**为什么需要 `is_searchable` 冗余列**：HNSW 是"先按向量距离取 K 个，再套 WHERE 过滤"（post-filter）。如果 `status='published'` 之类的条件写在 JOIN 后的 chunk 表上，HNSW 会先取 30 个近邻、过滤后可能只剩 3 条——**这就是最典型的"检索结果偏差"**。把可检索标志冗余到向量表并做成 partial index，等于让索引里**只存可检索的向量**，post-filter 不再削减结果。再叠加 pgvector 0.8 的 iterative scan 兜底更细的过滤（如 `business_date = 今天`），召回率才真正稳定。冗余带来的代价是"需要保证同步"，因此配一个触发器 + 一条巡检 SQL（§7.1 D-03）。

**HNSW vs IVFFlat 选型依据**

| 对比项 | HNSW | IVFFlat |
| --- | --- | --- |
| 原理 | 多层邻近图，逐层贪心搜索 | 先 k-means 聚成 `lists` 个簇，查询只扫 `probes` 个簇 |
| 是否需要训练 | **不需要** | **需要**：建索引时用已有数据聚类，空表建索引等于无效 |
| 增量写入 | 直接插入即生效，召回不衰减 | 新数据只归入既有簇；数据分布漂移后召回下降，**需周期 REINDEX** |
| 构建耗时 / 内存 | 慢、吃内存（`maintenance_work_mem` 建议 ≥ 1GB） | 快、省内存 |
| 查询延迟 | 更低（同召回率下） | 略高 |
| 召回率 | 高，`ef_search` 单调可调 | 依赖 `lists`/`probes` 配比与聚类质量 |
| 索引体积 | 大（约为向量数据的 1–2 倍） | 小 |
| 参数 | `m`（每层连接数，16 默认）、`ef_construction`（构建质量，128–200） | `lists ≈ sqrt(rows)`（< 100 万行）、`probes ≈ lists/10` |

| 场景 | 选择 | 理由 |
| --- | --- | --- |
| **本项目主路径**（7 个域，单域 < 5 万，菜单日更） | **HNSW** | 免训练 + 增量友好 + 召回稳定，正好匹配"每天 05:00 增量 upsert"的节奏；单域数据量小，构建耗时与内存都不是瓶颈 |
| `nutrition_knowledge` 未来扩到百万级公开成分库 | 可切 **IVFFlat** | 该域**批量导入、极少更新**，聚类漂移风险低；构建快、体积小的优势此时才体现 |
| PoC / 单元测试（几百行） | **不建索引**（顺序精确扫描） | 小数据量下精确检索 100% 召回，且可作为"索引召回率回测"的基准真值（见 §7.2 R-04） |
| 内存紧张、索引体积超预算 | HNSW on **halfvec** | `CREATE INDEX ... USING hnsw ((embedding::halfvec(1024)) halfvec_cosine_ops)`，索引体积约减半，召回损失通常 < 1pp；需同步改写查询的排序表达式 |

> **结论**：默认 HNSW，`m=16 / ef_construction=200 / ef_search=100`；IVFFlat 只作为"超大且静态的域"的备选，切换前必须跑黄金集回测对比 Recall@5。

**其余索引**

| 表 | 索引 | 类型 | 用途 |
| --- | --- | --- | --- |
| `kb.chunk` | `(domain, status, is_current) INCLUDE (chunk_id)` | btree | 过滤主路径 |
| | `(canteen, business_date, meal_period) WHERE is_current` | btree | 菜单类精确过滤 |
| | `tsv` | **GIN** | 全文/BM25 召回 |
| | `dish_tags` / `allergens` | GIN (`array_ops`) | 标签与过敏原数组包含查询 |
| | `attrs` | GIN (`jsonb_path_ops`) | 域特有字段查询 |
| | `content_sha256` | btree | 精确去重 |
| | `doc_id` | btree | 文档级级联操作 |
| `kb.chunk_embedding` | `embedding` | **HNSW partial** | 向量召回（见上） |
| | `(model_key, is_searchable)` | btree | 巡检与切模型 |
| `kb.entity` | `(entity_type, name_norm)` | unique btree | 实体链接（精确） |
| | `name_norm gin_trgm_ops` | **GIN trgm** | 实体链接（模糊，口语化菜名） |
| | `desc_embedding` | HNSW | 实体链接兜底（向量） |
| | `degree DESC` | btree | 超级节点识别与剪枝 |
| `kb.entity_alias` | `alias_norm` | btree + **GIN trgm** | 别名归一化 |
| `kb.relation` | `(head_id, relation_type) WHERE status='published'` | btree partial | 正向 1 跳展开 |
| | `(tail_id, relation_type) WHERE status='published'` | btree partial | 反向 1 跳展开 |
| `kb.chunk_entity` | `(entity_id, salience DESC)` | btree | 实体 → 高显著度 chunk |
| `kb.community` | `summary_embedding` | HNSW | global search |
| `kb.retrieval_log` | `(created_at)`、`(is_empty) WHERE is_empty` | btree | 偏差检测看板 |

---

## 5. 检索链路

### 5.1 端到端步骤与表交互

| # | 步骤 | 输入 | 交互的表/视图 | 输出 |
| --- | --- | --- | --- | --- |
| ① | 查询改写 | `query_sanitized` + 会话上下文 | —（轻量模型/规则） | 改写后问题 + 时间归一（`business_date`）+ 抽取的实体字面 |
| ② | 实体链接 | 实体字面 | `kb.entity_alias`（精确 → trgm ≥ 0.45）→ `kb.entity`（`name_norm`）→ `kb.entity.desc_embedding`（向量兜底） | `seed_entities uuid[]`；未命中写 `kb.alias_miss` |
| ③ | 域路由 | 意图 + 命中实体类型 | `kb.domain` 枚举（分区裁剪） | `domains kb.domain[]`（1–2 个） |
| ④ | 问题向量化 | 改写后问题 | —（同 `model_key` 的 embedding 服务） | `qvec vector(1024)`，**同样 L2 归一化** |
| ⑤a | Dense 召回 | `qvec`, `domains`, 过滤条件 | `kb.chunk_embedding`（HNSW）JOIN `kb.v_searchable_chunk` | Top-30 `(chunk_id, rank, cos_sim)` |
| ⑤b | Sparse 召回 | 改写后问题 | `kb.v_searchable_chunk.tsv`（GIN） | Top-30 `(chunk_id, rank, ts_rank)` |
| ⑤c | Graph 召回 | `seed_entities` | `kb.v_graph_edge`（递归 CTE）→ `kb.chunk_entity` → `kb.v_searchable_chunk` | Top-20 `(chunk_id, rank, path_score, path)` |
| ⑥ | RRF 融合 | 三路结果 | —（同一条 SQL 内的 CTE） | 候选 ≤ 60 条，带 `fused_score` |
| ⑦ | Rerank | 候选 + 问题 | —（Cross-Encoder） | Top-8 |
| ⑧ | MMR + 阈值 | Top-8 | — | Top-5，`cos_sim < 0.35` 丢弃 |
| ⑨ | 权威源去重 | Top-5 | `chunk.authority` | 同 `canteen+category` 只留 `primary` |
| ⑩ | 组装上下文 | Top-5 + 图谱路径 | `kb.relation_evidence`（补边证据） | `retrieved_docs[]`（含 `chunk_ref` / `verified_date` / `source_path`） |
| ⑪ | 记录 | 全过程 | `kb.retrieval_log` | 用于偏差检测与回归 |

**默认参数**

| 参数 | 值 | 说明 |
| --- | --- | --- |
| `top_k_dense` | 30 | 向量召回 |
| `top_k_sparse` | 30 | 全文召回 |
| `top_k_graph` | 20 | 图增强召回 |
| `graph_max_hops` | **2** | 超过 2 跳噪声急剧上升（见 §7.3 R-11） |
| `graph_decay` | 0.6 / 跳 | 路径分数衰减 |
| `rrf_k` | 60 | RRF 常数 |
| `w_dense / w_sparse / w_graph` | 1.0 / 0.8 / 0.6 | 融合权重 |
| `top_n_rerank` | 8 | 精排输入 |
| `top_k_final` | 5 | 送入 LLM |
| `score_threshold` | 0.35 | 低于此视为无召回 |
| `mmr_lambda` | 0.7 | 相关性 vs 多样性 |
| `hnsw.ef_search` | 100 | 可按域调整（`canteen_faq` 可降到 64） |
| `status_filter` | `published` | **硬过滤，视图层强制，不可关闭** |

### 5.2 混合检索 + 图增强的完整 SQL（Local Search）

```sql
-- 参数：$1 qvec, $2 查询文本, $3 domains, $4 canteen, $5 business_date,
--       $6 seed_entities, $7 allowed_relation_types, $8 allergen_exclude
SET LOCAL hnsw.ef_search       = 100;
SET LOCAL hnsw.iterative_scan  = 'relaxed_order';
SET LOCAL hnsw.max_scan_tuples = 20000;

WITH RECURSIVE
-- ① 图展开：从种子实体出发，最多 2 跳，带路径防环与超级节点剪枝
hop AS (
    SELECT e.entity_id,
           0                       AS depth,
           1.0::numeric            AS path_score,
           ARRAY[e.entity_id]      AS path,
           ARRAY[]::text[]         AS rel_path
    FROM   kb.entity e
    WHERE  e.entity_id = ANY ($6) AND NOT e.deleted AND e.status = 'published'
  UNION ALL
    SELECT g.dst,
           h.depth + 1,
           h.path_score * g.weight * g.confidence * 0.6,
           h.path || g.dst,
           h.rel_path || g.relation_type
    FROM   hop h
    JOIN   kb.v_graph_edge g ON g.src = h.entity_id
    JOIN   kb.entity te      ON te.entity_id = g.dst
    WHERE  h.depth < 2
      AND  g.relation_type = ANY ($7)
      AND  NOT g.dst = ANY (h.path)      -- 防环
      AND  te.degree <= 500              -- 超级节点剪枝，避免"米饭"这类节点炸开
      AND  h.path_score * g.weight > 0.10
),
-- ② Dense：向量召回（分区裁剪 + partial index 命中）
dense AS (
    SELECT v.chunk_id, v.domain,
           row_number() OVER (ORDER BY em.embedding <=> $1) AS rnk,
           1 - (em.embedding <=> $1)                        AS cos_sim
    FROM   kb.chunk_embedding em
    JOIN   kb.v_searchable_chunk v
           ON v.chunk_id = em.chunk_id AND v.domain = em.domain
    WHERE  em.is_searchable
      AND  em.model_key = 'bge-m3@v1.5'
      AND  em.domain    = ANY ($3)
      AND  ($4 IS NULL OR v.canteen = $4)
      AND  ($5 IS NULL OR v.business_date IS NULL OR v.business_date = $5)
      AND  ($8 IS NULL OR NOT (v.allergens && $8))   -- 过敏原硬排除
    ORDER  BY em.embedding <=> $1
    LIMIT  30
),
-- ③ Sparse：全文召回，解决"香煎鸡胸饭""3 号窗口"这类精确字面
sparse AS (
    SELECT v.chunk_id, v.domain,
           row_number() OVER (ORDER BY ts_rank_cd(v.tsv, q.query) DESC) AS rnk,
           ts_rank_cd(v.tsv, q.query)                                    AS ts_score
    FROM   kb.v_searchable_chunk v,
           websearch_to_tsquery('kb.zh'::regconfig, $2) AS q(query)
    WHERE  v.tsv @@ q.query
      AND  v.domain = ANY ($3)
      AND  ($4 IS NULL OR v.canteen = $4)
      AND  ($5 IS NULL OR v.business_date IS NULL OR v.business_date = $5)
      AND  ($8 IS NULL OR NOT (v.allergens && $8))
    ORDER  BY ts_score DESC
    LIMIT  30
),
-- ④ Graph：命中实体所在的 chunk（含 1–2 跳邻居的 chunk）
graph AS (
    SELECT v.chunk_id, v.domain,
           row_number() OVER (ORDER BY max(h.path_score * ce.salience) DESC) AS rnk,
           max(h.path_score * ce.salience) AS graph_score,
           (array_agg(DISTINCT h.rel_path))[1] AS rel_path
    FROM   hop h
    JOIN   kb.chunk_entity ce ON ce.entity_id = h.entity_id
    JOIN   kb.v_searchable_chunk v
           ON v.chunk_id = ce.chunk_id AND v.domain = ce.domain
    WHERE  ce.confidence >= 0.70
      AND  ($8 IS NULL OR NOT (v.allergens && $8))
    GROUP  BY v.chunk_id, v.domain
    ORDER  BY graph_score DESC
    LIMIT  20
),
-- ⑤ RRF 融合：只用排名，不用原始分数，天然跨模态可比
fused AS (
    SELECT chunk_id, domain,
           sum(w) AS fused_score,
           max(cos_sim)     AS cos_sim,
           max(ts_score)    AS ts_score,
           max(graph_score) AS graph_score,
           max(rel_path)    AS rel_path
    FROM (
        SELECT chunk_id, domain, 1.0 / (60 + rnk) * 1.0 AS w,
               cos_sim, NULL::real AS ts_score, NULL::numeric AS graph_score,
               NULL::text[] AS rel_path                       FROM dense
        UNION ALL
        SELECT chunk_id, domain, 1.0 / (60 + rnk) * 0.8,
               NULL, ts_score, NULL, NULL                     FROM sparse
        UNION ALL
        SELECT chunk_id, domain, 1.0 / (60 + rnk) * 0.6,
               NULL, NULL, graph_score, rel_path               FROM graph
    ) u
    GROUP BY chunk_id, domain
)
SELECT f.chunk_id, f.domain, v.chunk_ref, v.doc_id, v.title, v.breadcrumb, v.content,
       v.authority, v.verified_date, v.source_path, v.canteen, v.floor, v.price,
       f.fused_score, f.cos_sim, f.ts_score, f.graph_score, f.rel_path,
       CASE WHEN f.graph_score IS NOT NULL THEN 'graph_enhanced' ELSE 'vector_or_text' END AS recall_via
FROM   fused f
JOIN   kb.v_searchable_chunk v ON v.chunk_id = f.chunk_id AND v.domain = f.domain
-- 权威源去重：同一 canteen+category 若同时命中 primary 与 secondary，只留 primary
WHERE  NOT EXISTS (
         SELECT 1 FROM fused f2
         JOIN kb.v_searchable_chunk v2 ON v2.chunk_id = f2.chunk_id AND v2.domain = f2.domain
         WHERE v2.canteen  IS NOT DISTINCT FROM v.canteen
           AND v2.category IS NOT DISTINCT FROM v.category
           AND v2.authority = 'primary' AND v.authority <> 'primary'
       )
ORDER  BY f.fused_score DESC
LIMIT  8;   -- 交给 Rerank
```

**为什么 RRF 而不是加权归一化分数**：余弦相似度、`ts_rank_cd`、图路径分数三者量纲完全不同，直接加权需要在线标定，且分布随语料变化漂移。RRF 只吃**排名**，鲁棒且无需标定，是混合检索的工业标准做法。

### 5.3 图谱如何"补齐"纯向量召回不到的证据

以 `"一食堂二楼有没有不辣的鸡胸肉，我对花生过敏"` 为例：

| 召回路 | 命中内容 | 单独用它的问题 |
| --- | --- | --- |
| Dense | 「一食堂二楼轻食窗口菜品表」（语义相近） | 花生过敏信息不在这块，会漏 |
| Sparse | 「香煎鸡胸饭」所在行（字面精确） | 只有菜名与价格，无风险信息 |
| **Graph** | `dish:香煎鸡胸饭 --CONTAINS_ALLERGEN--> allergen:花生`（0 跳→1 跳）连带 `food_safety` 域的《花生交叉污染防控》chunk | — |

图增强的价值在于：**过敏原信息与菜品信息物理上在两个不同域的文档里**，纯向量检索必须"一次问题同时命中两个域"才能拼齐，实际很难；而图谱把 `dish → allergen → hazard` 的路径显式存下来了，一跳就把 `food_safety` 的证据 chunk 拉进候选集。

同理：

| 问法 | 关键路径 | 补齐的证据 |
| --- | --- | --- |
| "二楼有哪些窗口" | `canteen --HAS_WINDOW--> window --ON_FLOOR--> floor:2` | 楼层布局 + 各窗口介绍（跨多块） |
| "15 块能吃啥" | `budget_tier:10_15 <--IN_BUDGET_TIER-- dish --SERVES-- window` | 预算档整表 + 各菜品所在窗口 |
| "刷卡还是扫码" | `canteen --GOVERNED_BY--> policy:payment` | 规章制度对应条款（权威源 primary） |
| "有清真的吗" | `window[is_halal] --SERVES--> dish` | 清真窗口 + 其菜品 |
| "这个菜没了有替代吗" | `dish --SUBSTITUTE_OF--> dish` | 同类替代菜卡 |

### 5.4 全局型问题：Community / Global Search

判定规则（Planner 侧）：问题**不指向具体实体**且带有"哪些/都有什么/对比/推荐/整体"等聚合语义时，走 global search。

```sql
-- ① 用问题向量召回相关社区摘要（level 由问题粒度决定：对比类取 1，全局类取 2）
SELECT c.community_id, c.level, c.title, c.summary,
       1 - (c.summary_embedding <=> $1) AS sim
FROM   kb.community c
WHERE  c.is_current AND c.level = $2
ORDER  BY c.summary_embedding <=> $1
LIMIT  3;

-- ② 对命中社区取成员实体的高显著度证据 chunk（map 阶段的取证）
SELECT DISTINCT v.chunk_ref, v.content, v.verified_date, v.source_path, e.canonical_name
FROM   kb.community_member m
JOIN   kb.entity e        ON e.entity_id = m.entity_id
JOIN   kb.chunk_entity ce ON ce.entity_id = m.entity_id AND ce.salience >= 0.60
JOIN   kb.v_searchable_chunk v
       ON v.chunk_id = ce.chunk_id AND v.domain = ce.domain
WHERE  m.community_id = ANY ($3)
ORDER  BY v.chunk_ref
LIMIT  20;
```

然后由 LLM 做 map-reduce：每个社区摘要 + 其证据 chunk 生成局部答案 → 汇总为全局答案。**摘要本身不得作为价格/时间的事实来源**（摘要是二次加工），涉及具体数字时必须回落到 ② 拿到的原始 chunk。

### 5.5 域路由表（意图 → domain）

| 意图 | domain | 是否启用图增强 | 备注 |
| --- | --- | --- | --- |
| 食堂在哪 / 有几个食堂 / 楼层布局 | `canteen_profile` | ✅（`LOCATED_IN`/`ON_FLOOR`/`HAS_WINDOW`） | — |
| 营业时间 / 支付 / 饭卡 / 失物招领 / 投诉 / 政策 | `canteen_rules` | ✅（`GOVERNED_BY`） | 营业时间以 `03` 为 primary |
| 常见问题 | `canteen_faq` | ✅（`ANSWERS`） | — |
| 菜品介绍 / 某窗口有什么 / 停供通知 | `canteen_menu_docs` | ✅（`SERVES`/`AVAILABLE_AT`） | 实时价格与余量以 T1 为准 |
| 特色菜 / 按预算推荐 | `canteen_dish_features` | ✅（`IN_BUDGET_TIER`/`SUBSTITUTE_OF`） | S1 消费 |
| 营养 / 成分 | `nutrition_knowledge` | ✅（`HAS_NUTRIENT`） | S2 消费 |
| 过敏 / 异物 / 变质 | `food_safety` | ✅（`CONTAINS_ALLERGEN`/`MAY_CONTAIN`/`WARNS_ABOUT`） | **高风险，必注入提示** |
| 全局对比 / "都有什么" | 走 §5.4 global search | ✅（社区） | 数字须回落原始 chunk |
| 天气 / 拥挤度 / 余额 | **不检索** | — | 走 T4 / T2；无对应域 |

`rag_retrieve` action 未显式指定 `domains` 时由 Planner 选 1–2 个，禁止全库广播（既慢又引噪声）。

---

## 6. 字段与表结构的输入输出衔接

### 6.1 主数据流上的字段传递（一张表看全链路）

| 字段 | 产生阶段 | 写入位置 | 下游消费者 | 一致性约束 |
| --- | --- | --- | --- | --- |
| `source_path` | ① 快照 | `source_document.source_path` | 溯源展示、`reject_log`、去重判定 | 与知识库相对路径**逐字符一致**，禁止改写大小写 |
| `content_sha256`（文档级） | ① 快照 | `source_document.content_sha256` | 阶段 ⑧ 判断"文档是否变更"，未变则整篇跳过 | sha256(原文字节) |
| `doc_id` | ② 解析 | `source_document.doc_id` | `chunk.doc_id`（FK）、引用锚点前半段 | `doc_<sha1(source_path)[0:10]>`，**跨版本稳定** |
| `status` / `verified_date` | ② front_matter | `source_document` → 冗余到 `chunk` | ④ 门控、`v_searchable_chunk` 过滤、回答中的"最后核验" | `CHECK(status<>'published' OR verified_date IS NOT NULL)` 双表都建 |
| `domain` | ② 由目录映射 | `source_document.domain` → `chunk.domain`（分区键）→ `chunk_embedding.domain` | 域路由、分区裁剪 | 三表必须一致；由 `chunk` 的 FK + 应用层保证 |
| `seq` / `chunk_ref` | ⑤ 切分 | `chunk.seq` / `chunk.chunk_ref` | 引用锚点、出站护栏校验、`eval_golden` 标注 | `chunk_ref = doc_id || '::chunk_' || lpad(seq,3,'0')`，`UNIQUE(doc_id,seq,domain)` |
| `breadcrumb` | ⑤ 切分 | `chunk.breadcrumb` | **同时进入** `tsv` 生成列与 embedding 输入文本 | 检索期查询构造必须与入库时同一拼接方式 |
| `content` / `content_sha256`（块级） | ⑤⑦ | `chunk.content` / `chunk.content_sha256` | ⑨ embedding 输入、⑦ 去重、Rerank 输入、上下文组装 | 块级 sha256 决定"是否需要重算向量" |
| 归一化列（`canteen/floor/meal_period/spice_level/price/budget_tier/allergens`） | ⑥ 归一化 | `chunk.*`（强类型列） | 检索 WHERE 过滤、过敏原硬排除、图谱实体链接 | `CHECK` 约束兜底；无法识别置 `NULL` + 记 `reject_log`，**禁止猜默认值** |
| `attrs` | ⑥ 富化 | `chunk.attrs jsonb` | 低频域特有过滤（GIN） | 仅放"不需要建独立索引"的字段 |
| `embed_state` | ⑧⑨ | `chunk.embed_state` | `v_searchable_chunk` 过滤（`='ready'`）、outbox 驱动 | 状态机见 §3.4 |
| `model_key` / `dim` / `embedding` | ⑨ 向量化 | `chunk_embedding.*` | ⑤a Dense 召回 | `CHECK(dim=1024 AND vector_dims=1024)`；检索期 `model_key` 必须与查询向量同源 |
| `is_searchable` | ⑨ + 触发器 | `chunk_embedding.is_searchable` | HNSW partial index 的谓词 | 由 `chunk` 的 `is_current/deleted/status/embed_state/verified_date/expire_date` 计算，触发器同步，巡检 D-03 校验 |
| `entity_id` / `name_norm` | ⑩ 图抽取 | `entity.*` | 实体链接、图展开起点 | `UNIQUE(entity_type,name_norm)` |
| `relation_id` + `evidence` | ⑩ 图抽取 | `relation` + `relation_evidence` | ⑤c Graph 召回、出站引用校验 | **无证据的边被 `v_graph_edge` 排除** |
| `chunk_entity.salience` | ⑩ 图抽取 | `chunk_entity.salience` | 图召回排序（`path_score × salience`） | 0–1，规则抽取给 1.0，模型抽取按置信度 |
| `community_id` / `summary_embedding` | ⑪ 社区 | `community.*` | §5.4 global search | `is_current` 切换，数字必须回落原始 chunk |
| `is_current` | ⑫ 发布 | `source_document` + `chunk` | 所有检索视图 | **只有全部 chunk `ready` 才切换**（原子发布） |
| `recall_at5` | ⑬ 校验 | `ingest_run.recall_at5` | 发布门禁、告警 | < 85% 阻断并可回滚 |
| 检索结果 | 在线 | `retrieval_log` | 偏差检测、周报、评测集回流 | 只存脱敏后的 query hash 与结构化条件 |

### 6.2 表间关联全景

```
kb.ingest_run ──1:N──► kb.source_document ──1:N──► kb.chunk ──1:1..N──► kb.chunk_embedding
     │                        │  (doc_id FK)        │ (chunk_id,domain FK)   （每 model_key 一行）
     │                        │                     │
     ├──1:N──► kb.reject_log  │                     ├──1:N──► kb.chunk_entity ──N:1──► kb.entity
     └──1:N──► kb.embedding_job ◄── outbox ──────────┘                                    │
                                                     └──1:N──► kb.relation_evidence       │
                                                                      │                   │
kb.graph_run ──1:N──► kb.community ──1:N──► kb.community_member ──N:1──┼───────────────────┤
                                                                      │                   │
                       kb.relation ──N:1──► kb.entity (head_id) ◄──────┘                   │
                            └────────N:1──► kb.entity (tail_id) ◄─────────────────────────┤
                                                                                          │
                       kb.entity_alias ──N:1──────────────────────────────────────────────┘
                       kb.alias_miss（未命中台账，运营补录后转为 entity_alias）
                       kb.entity_type_dict / kb.relation_type_dict（被 entity/relation 外键引用）
                       kb.retrieval_log / kb.eval_golden（引用 chunk_ref，弱关联，不建 FK）
```

**级联规则**

| 操作 | 级联行为 | 理由 |
| --- | --- | --- |
| 删除 `source_document` | `chunk` CASCADE → `chunk_embedding` / `chunk_entity` / `relation_evidence` CASCADE | 避免孤儿向量与孤儿证据 |
| 删除 `entity` | `relation` / `chunk_entity` / `entity_alias` CASCADE | 避免悬挂边 |
| 删除 `relation` | `relation_evidence` CASCADE | — |
| **文档下线** | **不物理删除**：`is_current=false` + `deleted=true`（软删除），30 天后物理清理 | 保留历史可追溯与快速回滚 |
| `retrieval_log` | 无 FK（chunk 可能已被清理） | 日志不应阻塞数据清理 |

> **`eval_golden` 用 `chunk_ref`（文本）而非 `chunk_id`（uuid）关联**：`chunk_id` 每次重建会变，而 `chunk_ref = doc_id::chunk_seq` 在内容结构不变时是稳定的。黄金集必须跨重建有效，否则每次入库都要重新标注。

---

## 7. 质量与风险：问题、检测手段与处理方案

三类风险分别编号：**D**（Dirty data，脏数据）、**R**（Retrieval/Index，索引失效与检索偏差）、**G**（Graph，图谱特有）。每项给出「检测手段 + 处理方案 + 频率」。可执行 SQL 见 docs/11 §7，此处给出关键片段。

### 7.1 脏数据类（D）

| 编号 | 问题 | 检测手段 | 处理方案 | 频率 |
| --- | --- | --- | --- | --- |
| D-01 | `published` 却缺 `verified_date` | **写入即拦截**：`ck_doc_verified` / `ck_chunk_verified` CHECK 约束；历史数据用巡检 SQL | 拦截即 `reject_log(reason_code='MISSING_VERIFIED_DATE')`，状态强制降为 `draft`，不入检索 | 实时 + 每日 |
| D-02 | 归一化失败（辣度未识别、楼层越界、价格缺单位） | `CHECK` 约束 + `reject_log` 中 `stage='normalize'` 的 warn 计数 | 置 `NULL` 并记录，**禁止猜默认值**；周报推给运营补录 | 实时 + 周报 |
| D-03 | `is_searchable` 与 `chunk` 真实状态不一致（触发器漏触发/手工改数据） | 巡检 SQL 对比两表 | 立即执行 `kb.resync_searchable_flag()` 修复；若频繁出现，检查是否有绕过触发器的批量 UPDATE | 每小时 |
| D-04 | 孤儿向量（chunk 已删，向量残留） | `chunk_embedding LEFT JOIN chunk` 找 NULL | FK CASCADE 本应阻止；出现即说明有人 `DISABLE TRIGGER` 或直接操作分区，物理删除 + 审计 | 每日 |
| D-05 | 缺向量的 chunk（`embed_state='ready'` 但无 embedding 行） | 巡检 SQL | 打回 `pending` 并重新入队 `embedding_job` | 每小时 |
| D-06 | 向量维度/模型混用 | `SELECT model_key, dim, count(*) GROUP BY` 应只有一组 | 清理非当前 `model_key` 的行；切模型走 §8.3 双写流程 | 每日 |
| D-07 | 零向量 / NaN 向量（embedding 服务异常返回） | `vector_norm(embedding)` 异常检测 | 删除该行 + 打回 `pending`；worker 侧增加"范数必须在 0.99–1.01"的前置断言 | 每小时 |
| D-08 | 重复 chunk（同内容多份） | `content_sha256` 分组计数；`simhash` 汉明距离 ≤ 3 | 精确重复保留 `authority` 最高、`verified_date` 最新的一条，其余软删除；近似重复进人工复核队列 | 每日 |
| D-09 | 过期语料仍可检索 | `expire_date < CURRENT_DATE` 但仍出现在 `v_searchable_chunk` | 视图已含日期过滤；若命中说明视图被绕过 → 收回基表权限 | 每日 |
| D-10 | 空内容 / 超长块 | `CHECK(char_len BETWEEN 1 AND 1200)` | 写入时拦截，切分器修复后重跑 | 实时 |
| D-11 | 别名未命中（口语化菜名、食堂简称） | `kb.alias_miss` 按 `miss_text` 计数 Top-20 | 周报推运营补录到 `entity_alias`；补录后自动重跑相关查询回归 | 周报 |
| D-12 | 权威源冲突（营业时间两处不一致） | 同 `canteen+category` 存在 `primary` 与 `secondary` 且文本数字不同 | 检索层 primary 优先（已在 §5.2 SQL 内）；同时告警运营修正 `01` 概览摘要 | 每日 |

**关键检测 SQL**

```sql
-- D-03 冗余标志一致性（最容易被忽视、后果最严重的一致性问题）
SELECT count(*) AS mismatch
FROM   kb.chunk_embedding em
JOIN   kb.chunk c ON c.chunk_id = em.chunk_id AND c.domain = em.domain
WHERE  em.is_searchable <> (c.is_current AND NOT c.deleted
                            AND c.status = 'published' AND c.embed_state = 'ready'
                            AND c.verified_date IS NOT NULL
                            AND (c.expire_date IS NULL OR c.expire_date >= CURRENT_DATE));

-- D-05 ready 但无向量（检索会漏这些块，且不报错——静默故障）
SELECT c.domain, count(*) AS missing_embedding
FROM   kb.chunk c
LEFT   JOIN kb.chunk_embedding em
       ON em.chunk_id = c.chunk_id AND em.domain = c.domain
      AND em.model_key = 'bge-m3@v1.5'
WHERE  c.embed_state = 'ready' AND em.chunk_id IS NULL
GROUP  BY c.domain;

-- D-06 模型/维度混用
SELECT model_key, dim, count(*) FROM kb.chunk_embedding GROUP BY 1, 2 ORDER BY 3 DESC;

-- D-07 零向量与异常范数（已 L2 归一化，范数应恒为 1）
SELECT chunk_id, domain, vector_norm(embedding) AS norm
FROM   kb.chunk_embedding
WHERE  is_normalized AND (vector_norm(embedding) < 0.99 OR vector_norm(embedding) > 1.01);

-- D-08 近似重复（SimHash 汉明距离 ≤ 3；PG14+ 的 bit_count）
SELECT a.chunk_ref, b.chunk_ref,
       bit_count(a.simhash::bit(64) # b.simhash::bit(64)) AS hamming
FROM   kb.chunk a
JOIN   kb.chunk b ON a.domain = b.domain AND a.chunk_id < b.chunk_id
WHERE  a.simhash IS NOT NULL AND b.simhash IS NOT NULL
  AND  bit_count(a.simhash::bit(64) # b.simhash::bit(64)) <= 3
LIMIT  100;
```

### 7.2 索引失效与性能类（R-01 ~ R-08）

| 编号 | 问题 | 检测手段 | 处理方案 | 频率 |
| --- | --- | --- | --- | --- |
| R-01 | **HNSW 索引没被用上**（走了顺序扫描），延迟从 20ms 涨到 2s | `EXPLAIN (ANALYZE, BUFFERS)` 确认是否 `Index Scan using idx_emb_*_hnsw`；`pg_stat_user_indexes.idx_scan = 0` | 常见根因：① 查询 WHERE 与 partial index 谓词不匹配（必须显式写 `is_searchable AND model_key='...'`）；② `ORDER BY` 表达式与索引算子不一致（cosine 索引必须用 `<=>`）；③ 统计信息陈旧 → `ANALYZE`；④ 排序键被包在函数里 | 每次发版 + 每日 |
| R-02 | **过滤条件严格导致召回不足**（返回条数远少于 LIMIT） | 回测：同一查询开/关索引对比结果集大小与 Recall@5 | ① `is_searchable` partial index（已做）；② 开启 `hnsw.iterative_scan='relaxed_order'`；③ 调高 `hnsw.ef_search`；④ 极端选择性场景（如"某窗口某天"）直接退化为**精确扫描小分区**（数据量小于 5k 时更快也更准） | 每次索引变更 |
| R-03 | 新增分区忘记建 HNSW 索引 | 巡检：每个 `chunk_embedding` 分区是否存在 `amname='hnsw'` 的索引 | 建索引脚本纳入迁移；巡检报警 | 每日 |
| R-04 | 索引召回率退化（无声无息） | **黄金集回测**：`kb.eval_golden` 对每个 q 记录期望 `chunk_ref`，比较索引结果与"关索引的精确结果" | Recall@5 < 85% 告警；先调 `ef_search`，无效则 `REINDEX CONCURRENTLY`，仍无效则复核 embedding 模型版本 | 每次入库 + 每日 |
| R-05 | 换 embedding 模型后向量空间混用（旧向量与新查询向量比距离，结果全乱） | `model_key` 单值断言（D-06）+ 检索 SQL 强制 `model_key = $current` | 切模型走 §8.3：新 `model_key` 双写 → 回测 → 切换配置 → 清理旧行；**严禁原地覆盖** | 切模型时 |
| R-06 | 索引膨胀 / 大量软删除后残留 | `pg_relation_size(index) / pg_relation_size(table)` 趋势；`pg_stat_user_tables.n_dead_tup` | `VACUUM (ANALYZE)`；比例异常时 `REINDEX INDEX CONCURRENTLY` | 每周 |
| R-07 | 索引构建耗时过长 / OOM | 构建时长与 `maintenance_work_mem` 监控 | `SET maintenance_work_mem='2GB'`、`max_parallel_maintenance_workers=4`；仍不够则改用 `halfvec` 表达式索引减半体积 | 重建时 |
| R-08 | `tsvector` 生成列与分词配置不同步（改了 `kb.zh` 映射，旧行 tsv 未更新） | 抽样对比 `tsv` 与 `to_tsvector('kb.zh', breadcrumb\|\|' '\|\|content)` | 生成列不会自动重算 → 必须 `UPDATE kb.chunk SET content = content`（触发重算）或整表重建；把"改分词配置"列为需要全量重跑的变更 | 改配置时 |

**关键检测 SQL**

```sql
-- R-01 索引是否被使用
SELECT relname, indexrelname, idx_scan, idx_tup_read
FROM   pg_stat_user_indexes
WHERE  schemaname = 'kb' AND indexrelname LIKE '%hnsw%'
ORDER  BY idx_scan;      -- idx_scan = 0 且已上线一段时间 → 查询没走索引

-- R-03 每个向量分区都必须有 HNSW 索引
SELECT c.relname AS partition
FROM   pg_class c
JOIN   pg_namespace n ON n.oid = c.relnamespace
WHERE  n.nspname = 'kb' AND c.relname LIKE 'chunk_embedding_p_%'
  AND  NOT EXISTS (
         SELECT 1 FROM pg_index i
         JOIN pg_class ic ON ic.oid = i.indexrelid
         JOIN pg_am  am  ON am.oid  = ic.relam
         WHERE i.indrelid = c.oid AND am.amname = 'hnsw');

-- R-04 黄金集 Recall@5（低于 0.85 阻断发布）
WITH hit AS (
  SELECT g.query_id,
         bool_or(r.chunk_ref = ANY (g.expected_chunk_refs)) AS hit
  FROM   kb.eval_golden g
  CROSS  JOIN LATERAL kb.search_topk(g.query_text, g.domains, 5) r
  GROUP  BY g.query_id)
SELECT round(avg(hit::int)::numeric, 4) AS recall_at5 FROM hit;
```

### 7.3 检索偏差类（R-09 ~ R-13）

| 编号 | 问题 | 检测手段 | 处理方案 | 频率 |
| --- | --- | --- | --- | --- |
| R-09 | 中文分词不准（`zhparser` 未装或词典缺食堂专有词），BM25 召回差 | 对比"有/无 sparse 路"的 Recall@5；抽查 `to_tsquery` 结果 | ① 给 `zhparser` 加自定义词典（菜名、窗口名、食堂别名）；② 装不上时降级：`simple` 配置 + `pg_trgm` 相似度召回（`content % $q`），并把 `w_sparse` 从 0.8 降到 0.5 | 每次语料变更 |
| R-10 | 长尾菜名/口语化查询召回为 0（"鸡胸饭""一餐二楼"） | `retrieval_log.is_empty` 比例；无召回问题 Top-20 | 别名表补录（D-11）+ trgm 兜底 + 查询改写扩展同义词 | 周报 |
| R-11 | **图展开跳数爆炸引入噪声**（2 跳后把"米饭""汤"这类通用节点全拉进来） | `retrieval_log` 中 `recall_via='graph_enhanced'` 的条目被最终引用的比例（图召回精确率） | ① 硬限 `depth < 2`；② `degree <= 500` 超级节点剪枝（已在 SQL 内）；③ `path_score` 阈值 0.10；④ 按意图限制 `allowed_relation_types`，不允许任意边类型展开 | 每周 |
| R-12 | 社区摘要过时（菜品换季后摘要还在说旧菜） | `community.created_at` 与最近 `ingest_run` 的时间差；摘要中的实体是否仍 `is_current` | 每日重算社区；摘要引用的实体若已下线，该社区标记 `stale` 并不参与召回 | 每日 |
| R-13 | 结果偏向某一域（如什么问题都召回菜单） | `retrieval_log` 按 `domain` 的命中分布对比意图分布 | 检查域路由规则；必要时对高频域施加 RRF 权重惩罚；根因常是"某域 chunk 数远多于其它域"，可对超大域单独调低 `w_dense` | 每周 |

**检索偏差看板 SQL**

```sql
-- 无召回率与图召回精确率（近 7 天）
SELECT date_trunc('day', created_at) AS d,
       count(*)                                              AS queries,
       round(avg(is_empty::int)::numeric, 4)                  AS empty_rate,
       round(avg(top1_score)::numeric, 4)                     AS avg_top1,
       round(avg(graph_hit_cited::int)::numeric, 4)           AS graph_precision
FROM   kb.retrieval_log
WHERE  created_at >= now() - interval '7 days'
GROUP  BY 1 ORDER BY 1 DESC;

-- 域分布异常（某域占比 > 70% 需复核路由规则）
SELECT unnest(domains) AS domain, count(*),
       round(100.0 * count(*) / sum(count(*)) OVER (), 1) AS pct
FROM   kb.retrieval_log
WHERE  created_at >= now() - interval '7 days'
GROUP  BY 1 ORDER BY 2 DESC;
```

### 7.4 图谱特有风险（G）

| 编号 | 问题 | 检测手段 | 处理方案 |
| --- | --- | --- | --- |
| G-01 | 无证据的边（模型抽出来但没有原文支撑）→ **新的幻觉来源** | `relation LEFT JOIN relation_evidence` 找 NULL | `v_graph_edge` 已排除；同时删除该边并降低对应 extractor 的信任度 |
| G-02 | 实体重复（"一食堂" 与 "第一食堂" 建成两个实体） | 同 `entity_type` 下 `name_norm` 的 trgm 相似度 > 0.8 的成对实体 | 合并实体（保留 canonical，另一个转为 `entity_alias`），并迁移其边与提及 |
| G-03 | 别名歧义（同一 `alias_norm` 指向多个实体） | `entity_alias` 按 `alias_norm` 分组计数 > 1 | 引入实体类型消歧（同类型冲突则需运营裁定）；检索期按上下文实体类型择一 |
| G-04 | 超级节点（度数过大，展开即噪声） | `entity ORDER BY degree DESC LIMIT 20` | 加入剪枝白名单（`degree <= 500` 已生效）；对"米饭""汤"这类通用实体标记 `attrs.generic=true` 并禁止作为展开中转 |
| G-05 | 悬挂边 / 自环 | FK + `ck_rel_noself` CHECK | 写入即拦截 |
| G-06 | 关系时效失效（换季菜品仍连着窗口） | `relation.valid_to < CURRENT_DATE` 但仍被召回 | `v_graph_edge` 已含日期过滤；入库时对菜单类边写入 `valid_to` |
| G-07 | `degree` 物化值漂移 | 对比 `entity.degree` 与实际边计数 | `kb.resync_entity_degree()` 修复，每日执行 |

```sql
-- G-01 无证据的边
SELECT r.relation_id, r.relation_type
FROM   kb.relation r
LEFT   JOIN kb.relation_evidence e ON e.relation_id = r.relation_id
WHERE  e.relation_id IS NULL AND NOT r.deleted;

-- G-03 别名歧义
SELECT alias_norm, count(DISTINCT entity_id) AS n, array_agg(DISTINCT entity_id)
FROM   kb.entity_alias GROUP BY 1 HAVING count(DISTINCT entity_id) > 1;

-- G-07 degree 漂移
SELECT e.entity_id, e.canonical_name, e.degree AS materialized, x.actual
FROM   kb.entity e
JOIN  (SELECT src AS entity_id, count(*) AS actual FROM kb.v_graph_edge GROUP BY 1) x
       USING (entity_id)
WHERE  e.degree <> x.actual;
```

### 7.5 巡检的组织方式

| 频率 | 内容 | 触发方式 | 失败动作 |
| --- | --- | --- | --- |
| 实时 | CHECK 约束、FK、触发器 | 数据库层 | 写入失败 → `reject_log` |
| 每小时 | D-03 / D-05 / D-07 | `pg_cron` 或外部调度 | 自动修复（`resync_*` / 重新入队），连续 3 次失败告警 P2 |
| 每日 03:00 | D-01/04/06/08/09/12、R-03/04、G-01/03/07 | `pg_cron` | 写入 `kb.health_check_log`，P1 项告警 |
| 每周 | R-06 膨胀、R-11/13 偏差、D-11 别名周报 | 调度 + 运营看板 | 生成待办工单（T3） |
| 每次入库后 | R-04 黄金集回测 | indexer 内联 | Recall@5 < 85% → 不发布（`is_current` 不切换）+ 告警 |
| 每次发版 | R-01 EXPLAIN 断言（集成测试） | CI | 未命中 HNSW → 阻断发布 |

> 所有巡检结果统一写入 `kb.health_check_log(check_code, severity, metric, detail, checked_at)`，由 `docs/09` 的告警规则消费；Prometheus 指标名见 §9 对齐表。

---

## 8. 运营与迁移

### 8.1 更新频率与数据保留

| 数据 | 更新频率 | 保留策略 |
| --- | --- | --- |
| `canteen_profile` | 学期级 | 软删除保留 |
| `canteen_menu_docs` | 每日 05:00 增量 upsert；`business_date` 过期 30 天后软删除 | 历史版本可回溯 |
| `canteen_rules` | 月级 | 软删除保留 |
| `canteen_faq` | 周级 | 软删除保留 |
| `canteen_dish_features` | 周级 | 软删除保留 |
| `nutrition_knowledge` | 季度 | 软删除保留 |
| `food_safety` | 月级 | 软删除保留 |
| `retrieval_log` | 实时 | **30 天滚动清理**（按 `created_at` 分日表或直接 delete） |
| 社区快照 | 每日全量重算 | 旧 `is_current=false` 保留 1 天 |
| 全库备份 | 每日 03:00 | 依赖 PG PITR，保留 30 天 |

### 8.2 从 ChromaDB 迁移步骤（一次性）

| 步骤 | 动作 | 校验 |
| --- | --- | --- |
| 1 | 建 `kb` schema、扩展、表、分区、索引（docs/11 全部 DDL） | `\dt kb.*` |
| 2 | 从 ChromaDB 导出 7 个 collection 的 `document` + `metadata`（embedding 可选） | 导出条数 = 源条数 |
| 3 | 原样入库 `source_document` + `chunk`（`embed_state='pending'`，不迁移旧向量） | `content_sha256` 一致 |
| 4 | 用新 `model_key` **重新向量化**（不复用旧向量，避免模型不一致） | 全部 `ready` |
| 5 | 图抽取 + 社区构建 | 实体/边数符合预期 |
| 6 | 黄金集回测：新库 Recall@5 ≥ 旧库（或 ≥ 85%） | 达标才切换 |
| 7 | 灰度切换 `rag_retrieve` 数据源 → 下线 ChromaDB | 双读一周无差异 |

> **不复用旧 embedding 的原因**：ChromaDB 里的向量可能是旧模型/旧维度，直接迁移会让"新旧向量空间混用"（R-05）在第一天就发生。宁可重建，换取一致。

### 8.3 切换 embedding 模型的规范流程

```
① 新 model_key 注册（如 bge-m3@v2.0）
② 双写：对全量 chunk 以新 model_key 生成第二份向量（不停老向量）
③ 回测：分别用新旧 model_key 跑黄金集，Recall@5 对比
④ 达标 → 切换配置中的 current_model_key → 清理旧 model_key 行
⑤ 未达标 → 回滚（仍用旧 key），记录原因
```

禁止"原地覆盖 embedding 列 + 换 key"，那会破坏历史可追溯与回滚能力。

### 8.4 权限模型

| 角色 | 权限 |
| --- | --- |
| `kb_ingest` | `kb` 下基表 INSERT/UPDATE/DELETE（indexer/worker 专用） |
| `kb_reader` | **仅** `kb.v_searchable_chunk` / `kb.v_graph_edge` 等视图 SELECT（应用检索专用） |
| `kb_ops` | 巡检函数、`resync_*`、`rollback_document`、`publish_document` 执行权 |
| `postgres` 超级用户 | 建表/迁移（严禁应用持有） |

> `kb_reader` 拿不到基表，等于把 `status='published'` / `verified_date IS NOT NULL` / `embed_state='ready'` 这些硬过滤**从"应用约定"升级为"数据库权限边界"**，这是杜绝"检索结果偏差"最硬的一层。

---

## 9. 与评测/监控/其他文档的对齐

### 9.1 指标对齐（对应 docs/09 §4.2）

| 原指标 | 改造后 |
| --- | --- |
| `rag_recall_total` / `rag_empty_total`（label: collection） | label 改为 `domain`（`kb.retrieval_log` 提供） |
| 新增 `rag_graph_hit_total` | 图增强命中数（`recall_via='graph_enhanced'`） |
| 新增 `kb_embed_pending_total` | `embed_state='pending'` 积压（outbox 健康） |
| 新增 `kb_embed_failed_total` | `embed_state='failed'`（embedding 服务告警） |
| 新增 `kb_hsnw_scan_total` | HNSW 索引命中（R-01 检测） |
| 新增 `kb_mismatch_total` | `is_searchable` 不一致计数（D-03） |
| 新增 `kb_orphan_embedding_total` | 孤儿向量（D-04） |
| 新增 `kb_graph_stale_community_total` | 过期社区（R-12） |
| 新增 `kb_reject_total`（label: reason_code） | 脏数据拒绝率（D 类） |

### 9.2 文档职责调整

| 文档 | 调整点 |
| --- | --- |
| `docs/00-项目结构.md` | `agent/rag/` 下的 `chroma_client.py`/`collections.py` 改为 `pg_client.py`/`kb_schema.py`；`deploy/chroma/` 改 `deploy/postgres/`；端口表 `chromadb` 行移除（Postgres 已存在） |
| `docs/02-总体架构.md` | 数据流全景图中 `ChromaDB(7 collections)` 改 `PostgreSQL(kb schema, pgvector)`；选型表中 ChromaDB 行改 pgvector；`rag_retrieve → ChromaDB` 改 `→ PostgreSQL(pgvector)` |
| `docs/03-...md` | `rag_retrieve` 的 Action 说明由"查 ChromaDB"改"查 PostgreSQL（kb schema）" |
| `docs/04/05/07/08/10`、`skills/*`、`agent/tools/*` | 所有 `ChromaDB` 口径改 `PostgreSQL(pgvector)`；L3 降级"向量召回"表述不变，但数据源改为 PG；`CHROMA_*` 配置项改 `PG_*` |
| `README.md` | 技术选型表、核心能力矩阵、目录结构同步 |

---

## 10. 落地 checklist（改造验收）

- [ ] `kb` schema 全部 DDL 在 PG16 + pgvector 0.8.6 上 `\i` 执行零报错
- [ ] 7 个 `chunk` / `chunk_embedding` 分区各有一个 partial HNSW 索引（R-03 巡检通过）
- [ ] `v_searchable_chunk` 视图过滤正确（写一条 `draft`、一条缺 `verified_date`、一条 `pending`，均不可见）
- [ ] 一条完整写入链路跑通：知识库文件 → 快照 → 切分 → 归一化 → 事务 A → 向量化 → 事务 B → 图抽取 → 发布 → 可检索
- [ ] 模拟 embedding 服务失败：chunk 落 `failed`，检索不可见，恢复后重新入队成功
- [ ] 混合检索 SQL 跑通，`EXPLAIN` 确认命中 HNSW（R-01）
- [ ] 黄金集 Recall@5 ≥ 85%（R-04）
- [ ] 图增强用例（花生过敏、二楼有什么、15 块能吃啥）召回优于纯向量基线
- [ ] 过敏原硬排除（`allergens && $8`）生效
- [ ] 全局型问题走 community global search，数字回落原始 chunk
- [ ] 巡检任务全部挂上 `pg_cron`，D/R/G 三类告警规则配置完成
- [ ] ChromaDB 双读一周无差异后下线

> 完整可执行 SQL 清单见 **[11-数据库Schema与SQL清单](11-数据库Schema与SQL清单.md)**。






