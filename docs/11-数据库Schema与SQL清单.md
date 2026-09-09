# 11 · 数据库 Schema 与 SQL 清单（可执行）

> 本文件是 `docs/06` 的可执行落地版：所有建库、索引、写入、检索、巡检 SQL 直接可用。目标环境 **PostgreSQL 16+，pgvector ≥ 0.8.0（建议 0.8.6）**。
>
> 执行顺序：§1 扩展 → §2 枚举/字典/运维表 → §3 主表与分区 → §4 索引 → §5 函数与触发器 → §6 检索 SQL → §7 巡检 SQL → §8 权限。

---

## 1. 扩展与 schema

```sql
CREATE SCHEMA IF NOT EXISTS kb;

CREATE EXTENSION IF NOT EXISTS vector;      -- pgvector
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS btree_gin;
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- 中文分词（zhparser 未安装时见 §9 降级方案）
CREATE TEXT SEARCH CONFIGURATION IF NOT EXISTS kb.zh (PARSER = zhparser);
ALTER  TEXT SEARCH CONFIGURATION kb.zh
       ADD MAPPING FOR n,v,a,i,e,l,j,t WITH simple;
```

---

## 2. 枚举、字典与运维表

```sql
-- ============ 枚举 ============
CREATE TYPE kb.domain AS ENUM (
  'canteen_profile', 'canteen_menu_docs', 'canteen_rules', 'canteen_faq',
  'canteen_dish_features', 'nutrition_knowledge', 'food_safety');
CREATE TYPE kb.corpus_status AS ENUM ('draft', 'published', 'expired');
CREATE TYPE kb.authority     AS ENUM ('primary', 'secondary', 'index');
CREATE TYPE kb.embed_state   AS ENUM ('pending', 'running', 'ready', 'failed');
CREATE TYPE kb.job_state     AS ENUM ('queued', 'processing', 'done', 'failed');
CREATE TYPE kb.ingest_status AS ENUM ('running', 'published', 'failed', 'rolled_back');

-- ============ 类型字典（可增量扩展，不用 ENUM）============
CREATE TABLE kb.entity_type_dict (
  type_code    text PRIMARY KEY,
  description  text NOT NULL,
  generic      boolean NOT NULL DEFAULT false,   -- 通用实体（米饭/汤），禁作展开中转
  created_at   timestamptz NOT NULL DEFAULT now()
);
INSERT INTO kb.entity_type_dict (type_code, description, generic) VALUES
  ('campus','校区',false), ('canteen','食堂',false), ('window','窗口',false),
  ('floor','楼层',false),  ('dish','菜品',false),     ('allergen','过敏原',false),
  ('nutrient','营养素',false), ('policy','规章制度',false),
  ('faq','常见问题',false),    ('hazard','风险主题',false),
  ('budget_tier','预算档',false), ('meal_period','餐次',false),
  ('tag','标签',false), ('generic_ingredient','通用食材',true);

CREATE TABLE kb.relation_type_dict (
  type_code   text PRIMARY KEY,
  description text NOT NULL,
  bidirectional boolean NOT NULL DEFAULT false,
  created_at  timestamptz NOT NULL DEFAULT now()
);
INSERT INTO kb.relation_type_dict (type_code, description, bidirectional) VALUES
  ('LOCATED_IN','位于',false),
  ('ON_FLOOR','位于楼层',false),
  ('HAS_WINDOW','拥有窗口',false),
  ('SERVES','供应',false),
  ('CONTAINS_ALLERGEN','含过敏原',false),
  ('MAY_CONTAIN','可能含',false),
  ('HAS_NUTRIENT','含营养素',false),
  ('IN_BUDGET_TIER','属于预算档',false),
  ('AVAILABLE_AT','供应时段',false),
  ('SUBSTITUTE_OF','可替代',true),
  ('GOVERNED_BY','受规章约束',false),
  ('ANSWERS','回答',false),
  ('WARNS_ABOUT','警示',false);

-- ============ 入库运行记录 ============
CREATE TABLE kb.ingest_run (
  run_id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  kb_snapshot_at  timestamptz NOT NULL,
  status          kb.ingest_status NOT NULL DEFAULT 'running',
  files_total     int NOT NULL DEFAULT 0,
  chunks_total    int NOT NULL DEFAULT 0,
  rejected_total  int NOT NULL DEFAULT 0,
  recall_at5      numeric(6,4),
  started_at      timestamptz NOT NULL DEFAULT now(),
  finished_at     timestamptz,
  operator        text NOT NULL DEFAULT 'system'
);

CREATE TABLE kb.graph_run (
  graph_run_id    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  is_current      boolean NOT NULL DEFAULT false,
  entities_total  int NOT NULL DEFAULT 0,
  relations_total int NOT NULL DEFAULT 0,
  communities     int NOT NULL DEFAULT 0,
  started_at      timestamptz NOT NULL DEFAULT now(),
  finished_at     timestamptz
);

-- ============ 拒绝入库台账（脏数据可追溯）============
CREATE TABLE kb.reject_log (
  reject_id   uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  run_id      uuid REFERENCES kb.ingest_run(run_id) ON DELETE CASCADE,
  source_path text NOT NULL,
  stage       text NOT NULL,          -- parse/gate/normalize/dedup
  reason_code text NOT NULL,          -- MISSING_VERIFIED_DATE / FLOOR_OUT_OF_RANGE / ...
  severity    text NOT NULL DEFAULT 'warn',   -- warn / error
  detail      jsonb,
  created_at  timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_reject_code ON kb.reject_log (reason_code, created_at DESC);

-- ============ 向量化 outbox 队列 ============
CREATE TABLE kb.embedding_job (
  job_id      uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  chunk_id    uuid NOT NULL,
  domain      kb.domain NOT NULL,
  model_key   text NOT NULL DEFAULT 'bge-m3@v1.5',
  state       kb.job_state NOT NULL DEFAULT 'queued',
  attempts    int NOT NULL DEFAULT 0,
  last_error  text,
  created_at  timestamptz NOT NULL DEFAULT now(),
  updated_at  timestamptz NOT NULL DEFAULT now(),
  UNIQUE (chunk_id, domain, model_key)
);
CREATE INDEX idx_embed_job_claim ON kb.embedding_job (state, created_at)
  WHERE state IN ('queued','failed');

-- ============ 别名未命中台账 ============
CREATE TABLE kb.alias_miss (
  miss_id    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  miss_text  text NOT NULL,
  hit_count  int NOT NULL DEFAULT 1,
  first_seen timestamptz NOT NULL DEFAULT now(),
  last_seen  timestamptz NOT NULL DEFAULT now(),
  resolved   boolean NOT NULL DEFAULT false
);
CREATE UNIQUE INDEX uq_alias_miss ON kb.alias_miss (miss_text);

-- ============ 检索日志（30 天滚动）============
CREATE TABLE kb.retrieval_log (
  log_id        uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  trace_id      text,
  thread_id     text,
  query_hash    char(64) NOT NULL,      -- 脱敏：只存 hash，不存明文
  domains       kb.domain[] NOT NULL,
  is_empty      boolean NOT NULL DEFAULT false,
  top1_score    real,
  graph_hit     boolean NOT NULL DEFAULT false,
  graph_hit_cited boolean NOT NULL DEFAULT false,
  latency_ms    int,
  created_at    timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_retr_time ON kb.retrieval_log (created_at DESC);
CREATE INDEX idx_retr_empty ON kb.retrieval_log (is_empty) WHERE is_empty;

-- ============ 检索黄金集 ============
CREATE TABLE kb.eval_golden (
  query_id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  query_text        text NOT NULL,
  domains           kb.domain[] NOT NULL,
  expected_chunk_refs text[] NOT NULL,
  note              text
);

-- ============ 巡检结果 ============
CREATE TABLE kb.health_check_log (
  check_code text NOT NULL,
  severity   text NOT NULL,             -- P0/P1/P2
  metric     numeric,
  detail     jsonb,
  checked_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_health ON kb.health_check_log (check_code, checked_at DESC);
```

---

## 3. 主表与分区

### 3.1 文档层

```sql
CREATE TABLE kb.source_document (
  doc_id          text PRIMARY KEY,
  domain          kb.domain NOT NULL,
  source_path     text NOT NULL,
  source_uri      text NOT NULL,
  title           text NOT NULL,
  doc_version     int NOT NULL DEFAULT 1,
  content_sha256  char(64) NOT NULL,
  campus          text,
  canteen         text,
  category        text,
  authority       kb.authority NOT NULL DEFAULT 'primary',
  status          kb.corpus_status NOT NULL,
  verified_date   date,
  effective_date  date,
  expire_date     date,
  reviewer        text,
  reviewed_at     date,
  acl             text[] NOT NULL DEFAULT '{student,staff,public}',
  risk_level      text NOT NULL DEFAULT 'normal',
  kb_snapshot_at  timestamptz NOT NULL,
  ingest_run_id   uuid NOT NULL REFERENCES kb.ingest_run(run_id),
  is_current      boolean NOT NULL DEFAULT false,
  deleted         boolean NOT NULL DEFAULT false,
  created_at      timestamptz NOT NULL DEFAULT now(),
  updated_at      timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT uq_doc_path_ver UNIQUE (source_path, doc_version),
  CONSTRAINT ck_doc_verified CHECK (status <> 'published' OR verified_date IS NOT NULL),
  CONSTRAINT ck_doc_dates CHECK (expire_date IS NULL OR effective_date IS NULL
                                 OR expire_date >= effective_date)
);
CREATE UNIQUE INDEX uq_doc_current ON kb.source_document (source_path)
  WHERE is_current AND NOT deleted;
CREATE INDEX idx_doc_domain ON kb.source_document (domain, is_current) WHERE is_current;
```

### 3.2 文本层（按 domain 分区）

```sql
CREATE TABLE kb.chunk (
  chunk_id        uuid NOT NULL DEFAULT gen_random_uuid(),
  domain          kb.domain NOT NULL,
  doc_id          text NOT NULL,
  seq             int NOT NULL,
  chunk_ref       text NOT NULL,
  breadcrumb      text NOT NULL,
  content         text NOT NULL,
  content_sha256  char(64) NOT NULL,
  simhash         bigint,
  char_len        int NOT NULL,
  token_len       int NOT NULL,
  status          kb.corpus_status NOT NULL,
  authority       kb.authority NOT NULL DEFAULT 'primary',
  verified_date   date,
  effective_date  date,
  expire_date     date,
  campus          text,
  canteen         text,
  window_name     text,
  floor           smallint,
  business_date   date,
  meal_period     text,
  spice_level     smallint,
  price           numeric(6,2),
  budget_tier     text,
  dish_tags       text[] NOT NULL DEFAULT '{}',
  allergens       text[] NOT NULL DEFAULT '{}',
  hazard_type     text,
  attrs           jsonb NOT NULL DEFAULT '{}'::jsonb,
  tsv             tsvector GENERATED ALWAYS AS
                    (to_tsvector('kb.zh'::regconfig, breadcrumb || ' ' || content)) STORED,
  embed_state     kb.embed_state NOT NULL DEFAULT 'pending',
  is_current      boolean NOT NULL DEFAULT false,
  deleted         boolean NOT NULL DEFAULT false,
  ingest_run_id   uuid NOT NULL,
  created_at      timestamptz NOT NULL DEFAULT now(),
  updated_at      timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (chunk_id, domain),
  CONSTRAINT uq_chunk_doc_seq UNIQUE (doc_id, seq, domain),
  FOREIGN KEY (doc_id) REFERENCES kb.source_document(doc_id) ON DELETE CASCADE,
  CONSTRAINT ck_chunk_floor  CHECK (floor       IS NULL OR floor       BETWEEN 1 AND 2),
  CONSTRAINT ck_chunk_spice  CHECK (spice_level IS NULL OR spice_level BETWEEN 0 AND 3),
  CONSTRAINT ck_chunk_price  CHECK (price       IS NULL OR price > 0),
  CONSTRAINT ck_chunk_meal   CHECK (meal_period IS NULL OR
                                    meal_period IN ('breakfast','lunch','dinner')),
  CONSTRAINT ck_chunk_tier   CHECK (budget_tier IS NULL OR
                                    budget_tier IN ('le_10','10_15','gt_15')),
  CONSTRAINT ck_chunk_len    CHECK (char_len BETWEEN 1 AND 1200),
  CONSTRAINT ck_chunk_verified CHECK (status <> 'published' OR verified_date IS NOT NULL)
) PARTITION BY LIST (domain);

CREATE TABLE kb.chunk_p_menu    PARTITION OF kb.chunk FOR VALUES IN ('canteen_menu_docs');
CREATE TABLE kb.chunk_p_rules   PARTITION OF kb.chunk FOR VALUES IN ('canteen_rules');
CREATE TABLE kb.chunk_p_profile PARTITION OF kb.chunk FOR VALUES IN ('canteen_profile');
CREATE TABLE kb.chunk_p_faq     PARTITION OF kb.chunk FOR VALUES IN ('canteen_faq');
CREATE TABLE kb.chunk_p_feature PARTITION OF kb.chunk FOR VALUES IN ('canteen_dish_features');
CREATE TABLE kb.chunk_p_nutri   PARTITION OF kb.chunk FOR VALUES IN ('nutrition_knowledge');
CREATE TABLE kb.chunk_p_safety  PARTITION OF kb.chunk FOR VALUES IN ('food_safety');
```

### 3.3 向量层（按 domain 分区）

```sql
CREATE TABLE kb.chunk_embedding (
  chunk_id      uuid NOT NULL,
  domain        kb.domain NOT NULL,
  model_key     text NOT NULL,
  dim           smallint NOT NULL,
  embedding     vector(1024) NOT NULL,
  is_normalized boolean NOT NULL DEFAULT true,
  is_searchable boolean NOT NULL DEFAULT false,
  embedded_at   timestamptz NOT NULL DEFAULT now(),
  ingest_run_id uuid NOT NULL,
  PRIMARY KEY (chunk_id, domain, model_key),
  FOREIGN KEY (chunk_id, domain) REFERENCES kb.chunk(chunk_id, domain) ON DELETE CASCADE,
  CONSTRAINT ck_emb_dim CHECK (dim = 1024 AND vector_dims(embedding) = 1024)
) PARTITION BY LIST (domain);

CREATE TABLE kb.chunk_embedding_p_menu    PARTITION OF kb.chunk_embedding FOR VALUES IN ('canteen_menu_docs');
CREATE TABLE kb.chunk_embedding_p_rules   PARTITION OF kb.chunk_embedding FOR VALUES IN ('canteen_rules');
CREATE TABLE kb.chunk_embedding_p_profile PARTITION OF kb.chunk_embedding FOR VALUES IN ('canteen_profile');
CREATE TABLE kb.chunk_embedding_p_faq     PARTITION OF kb.chunk_embedding FOR VALUES IN ('canteen_faq');
CREATE TABLE kb.chunk_embedding_p_feature PARTITION OF kb.chunk_embedding FOR VALUES IN ('canteen_dish_features');
CREATE TABLE kb.chunk_embedding_p_nutri   PARTITION OF kb.chunk_embedding FOR VALUES IN ('nutrition_knowledge');
CREATE TABLE kb.chunk_embedding_p_safety  PARTITION OF kb.chunk_embedding FOR VALUES IN ('food_safety');
```

### 3.4 图层

```sql
CREATE TABLE kb.entity (
  entity_id      uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  entity_type    text NOT NULL REFERENCES kb.entity_type_dict(type_code),
  canonical_name text NOT NULL,
  name_norm      text NOT NULL,
  description    text,
  desc_embedding vector(1024),
  attrs          jsonb NOT NULL DEFAULT '{}'::jsonb,
  degree         int NOT NULL DEFAULT 0,
  status         kb.corpus_status NOT NULL DEFAULT 'published',
  confidence     numeric(4,3) NOT NULL DEFAULT 1.000,
  deleted        boolean NOT NULL DEFAULT false,
  created_at     timestamptz NOT NULL DEFAULT now(),
  updated_at     timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT uq_entity UNIQUE (entity_type, name_norm)
);
CREATE INDEX idx_entity_trgm ON kb.entity USING gin (name_norm gin_trgm_ops);
CREATE INDEX idx_entity_emb   ON kb.entity USING hnsw (desc_embedding vector_cosine_ops)
  WHERE desc_embedding IS NOT NULL;

CREATE TABLE kb.entity_alias (
  alias_id     uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  entity_id    uuid NOT NULL REFERENCES kb.entity(entity_id) ON DELETE CASCADE,
  alias_text   text NOT NULL,
  alias_norm   text NOT NULL,
  source       text NOT NULL DEFAULT 'manual',
  created_at   timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT uq_alias UNIQUE (alias_norm)
);
CREATE INDEX idx_alias_trgm ON kb.entity_alias USING gin (alias_norm gin_trgm_ops);

CREATE TABLE kb.relation (
  relation_id   uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  head_id       uuid NOT NULL REFERENCES kb.entity(entity_id) ON DELETE CASCADE,
  relation_type text NOT NULL REFERENCES kb.relation_type_dict(type_code),
  tail_id       uuid NOT NULL REFERENCES kb.entity(entity_id) ON DELETE CASCADE,
  weight        numeric(5,4) NOT NULL DEFAULT 1.0000,
  confidence    numeric(4,3) NOT NULL DEFAULT 1.000,
  valid_from    date NOT NULL DEFAULT DATE '1900-01-01',
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
CREATE INDEX idx_rel_head ON kb.relation (head_id, relation_type) WHERE status='published' AND NOT deleted;
CREATE INDEX idx_rel_tail ON kb.relation (tail_id, relation_type) WHERE status='published' AND NOT deleted;

CREATE TABLE kb.relation_evidence (
  relation_id uuid NOT NULL REFERENCES kb.relation(relation_id) ON DELETE CASCADE,
  chunk_id    uuid NOT NULL,
  domain      kb.domain NOT NULL,
  quote       text NOT NULL,
  extractor   text NOT NULL,
  confidence  numeric(4,3) NOT NULL DEFAULT 1.000,
  PRIMARY KEY (relation_id, chunk_id, domain),
  FOREIGN KEY (chunk_id, domain) REFERENCES kb.chunk(chunk_id, domain) ON DELETE CASCADE
);

CREATE TABLE kb.chunk_entity (
  chunk_id   uuid NOT NULL,
  domain     kb.domain NOT NULL,
  entity_id  uuid NOT NULL REFERENCES kb.entity(entity_id) ON DELETE CASCADE,
  mention    text NOT NULL,
  char_start int NOT NULL,
  char_end   int NOT NULL,
  salience   numeric(4,3) NOT NULL DEFAULT 0.500,
  extractor  text NOT NULL,
  confidence numeric(4,3) NOT NULL DEFAULT 1.000,
  PRIMARY KEY (chunk_id, domain, entity_id, char_start),
  FOREIGN KEY (chunk_id, domain) REFERENCES kb.chunk(chunk_id, domain) ON DELETE CASCADE
);
CREATE INDEX idx_ce_entity ON kb.chunk_entity (entity_id, salience DESC);

CREATE TABLE kb.community (
  community_id      uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  graph_run_id      uuid NOT NULL REFERENCES kb.graph_run(graph_run_id),
  level             smallint NOT NULL,        -- 0/1/2
  title             text NOT NULL,
  summary           text NOT NULL,
  summary_embedding vector(1024),
  entity_ids        uuid[] NOT NULL,
  is_current        boolean NOT NULL DEFAULT false,
  stale             boolean NOT NULL DEFAULT false,
  created_at        timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_community_emb ON kb.community USING hnsw (summary_embedding vector_cosine_ops)
  WHERE is_current AND NOT stale AND summary_embedding IS NOT NULL;

CREATE TABLE kb.community_member (
  community_id uuid NOT NULL REFERENCES kb.community(community_id) ON DELETE CASCADE,
  entity_id    uuid NOT NULL REFERENCES kb.entity(entity_id) ON DELETE CASCADE,
  PRIMARY KEY (community_id, entity_id)
);
```

---

## 4. 索引（向量 + 文本）

```sql
-- ============ 向量 HNSW（每个分区一个 partial index）============
-- 菜单域数据量最大：m=16, ef_construction=200；其余域可 128
CREATE INDEX idx_emb_menu_hnsw ON kb.chunk_embedding_p_menu
  USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 200)
  WHERE is_searchable AND model_key = 'bge-m3@v1.5';

CREATE INDEX idx_emb_rules_hnsw ON kb.chunk_embedding_p_rules
  USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 128)
  WHERE is_searchable AND model_key = 'bge-m3@v1.5';

CREATE INDEX idx_emb_profile_hnsw ON kb.chunk_embedding_p_profile
  USING hnsw (embedding vector_cosine_ops) WITH (m=16, ef_construction=128)
  WHERE is_searchable AND model_key = 'bge-m3@v1.5';
CREATE INDEX idx_emb_faq_hnsw ON kb.chunk_embedding_p_faq
  USING hnsw (embedding vector_cosine_ops) WITH (m=16, ef_construction=128)
  WHERE is_searchable AND model_key = 'bge-m3@v1.5';
CREATE INDEX idx_emb_feature_hnsw ON kb.chunk_embedding_p_feature
  USING hnsw (embedding vector_cosine_ops) WITH (m=16, ef_construction=128)
  WHERE is_searchable AND model_key = 'bge-m3@v1.5';
CREATE INDEX idx_emb_nutri_hnsw ON kb.chunk_embedding_p_nutri
  USING hnsw (embedding vector_cosine_ops) WITH (m=16, ef_construction=128)
  WHERE is_searchable AND model_key = 'bge-m3@v1.5';
CREATE INDEX idx_emb_safety_hnsw ON kb.chunk_embedding_p_safety
  USING hnsw (embedding vector_cosine_ops) WITH (m=16, ef_construction=128)
  WHERE is_searchable AND model_key = 'bge-m3@v1.5';

-- ============ 文本过滤 / 全文 / 数组 ============
CREATE INDEX idx_chunk_filter ON kb.chunk (domain, status, is_current) INCLUDE (chunk_id);
CREATE INDEX idx_chunk_canteen ON kb.chunk (canteen, business_date, meal_period) WHERE is_current;
CREATE INDEX idx_chunk_tsv    ON kb.chunk USING gin (tsv);
CREATE INDEX idx_chunk_tags   ON kb.chunk USING gin (dish_tags);
CREATE INDEX idx_chunk_allerg ON kb.chunk USING gin (allergens);
CREATE INDEX idx_chunk_attrs  ON kb.chunk USING gin (attrs jsonb_path_ops);
CREATE INDEX idx_chunk_sha    ON kb.chunk (content_sha256);
CREATE INDEX idx_chunk_doc    ON kb.chunk (doc_id);
```

---

## 5. 函数与触发器

### 5.1 检索视图（对外唯一出口）

```sql
CREATE OR REPLACE VIEW kb.v_searchable_chunk AS
SELECT c.chunk_id, c.domain, c.doc_id, c.chunk_ref, c.breadcrumb, c.content,
       c.authority, c.verified_date, c.campus, c.canteen, c.window_name, c.floor,
       c.business_date, c.meal_period, c.spice_level, c.price, c.budget_tier,
       c.dish_tags, c.allergens, c.hazard_type, c.attrs, c.tsv,
       d.title, d.source_path, d.category, d.reviewer, d.doc_version
FROM   kb.chunk c
JOIN   kb.source_document d USING (doc_id)
WHERE  c.is_current AND NOT c.deleted
  AND  c.status = 'published'
  AND  c.embed_state = 'ready'
  AND  c.verified_date IS NOT NULL
  AND  (c.expire_date IS NULL OR c.expire_date >= CURRENT_DATE)
  AND  d.is_current AND NOT d.deleted;

CREATE OR REPLACE VIEW kb.v_graph_edge AS
SELECT r.relation_id, r.head_id AS src, r.tail_id AS dst, r.relation_type,
       r.weight, r.confidence, false AS reversed
FROM   kb.relation r
WHERE  r.status='published' AND NOT r.deleted
  AND  (r.valid_to IS NULL OR r.valid_to >= CURRENT_DATE)
  AND  EXISTS (SELECT 1 FROM kb.relation_evidence e WHERE e.relation_id = r.relation_id)
UNION ALL
SELECT r.relation_id, r.tail_id, r.head_id, r.relation_type,
       r.weight * 0.9, r.confidence, true
FROM   kb.relation r
WHERE  r.status='published' AND NOT r.deleted
  AND  (r.valid_to IS NULL OR r.valid_to >= CURRENT_DATE)
  AND  EXISTS (SELECT 1 FROM kb.relation_evidence e WHERE e.relation_id = r.relation_id);
```

### 5.2 chunk upsert（阶段 ⑧ 核心）

```sql
CREATE OR REPLACE FUNCTION kb.upsert_chunk(
  p_doc_id text, p_domain kb.domain, p_seq int, p_chunk_ref text,
  p_breadcrumb text, p_content text, p_content_sha256 char(64), p_simhash bigint,
  p_char_len int, p_token_len int, p_status kb.corpus_status,
  p_authority kb.authority, p_verified_date date, p_campus text, p_canteen text,
  p_window_name text, p_floor smallint, p_business_date date, p_meal_period text,
  p_spice_level smallint, p_price numeric, p_budget_tier text,
  p_dish_tags text[], p_allergens text[], p_hazard_type text,
  p_attrs jsonb, p_ingest_run_id uuid
) RETURNS uuid AS $$
DECLARE v_chunk_id uuid;
BEGIN
  INSERT INTO kb.chunk (doc_id, domain, seq, chunk_ref, breadcrumb, content,
                        content_sha256, simhash, char_len, token_len,
                        status, authority, verified_date, campus, canteen,
                        window_name, floor, business_date, meal_period,
                        spice_level, price, budget_tier, dish_tags, allergens,
                        hazard_type, attrs, embed_state, is_current, ingest_run_id)
  VALUES (p_doc_id, p_domain, p_seq, p_chunk_ref, p_breadcrumb, p_content,
          p_content_sha256, p_simhash, p_char_len, p_token_len,
          p_status, p_authority, p_verified_date, p_campus, p_canteen,
          p_window_name, p_floor, p_business_date, p_meal_period,
          p_spice_level, p_price, p_budget_tier, p_dish_tags, p_allergens,
          p_hazard_type, p_attrs, 'pending', false, p_ingest_run_id)
  ON CONFLICT (doc_id, seq, domain) DO UPDATE SET
    content        = EXCLUDED.content,
    content_sha256 = EXCLUDED.content_sha256,
    breadcrumb     = EXCLUDED.breadcrumb,
    status         = EXCLUDED.status,
    verified_date  = EXCLUDED.verified_date,
    canteen        = EXCLUDED.canteen,
    floor          = EXCLUDED.floor,
    business_date  = EXCLUDED.business_date,
    meal_period    = EXCLUDED.meal_period,
    spice_level    = EXCLUDED.spice_level,
    price          = EXCLUDED.price,
    budget_tier    = EXCLUDED.budget_tier,
    dish_tags      = EXCLUDED.dish_tags,
    allergens      = EXCLUDED.allergens,
    attrs          = EXCLUDED.attrs,
    -- 内容未变 → 沿用原状态；内容变了 → 打回 pending 重新向量化
    embed_state    = CASE WHEN kb.chunk.content_sha256 = EXCLUDED.content_sha256
                          THEN kb.chunk.embed_state ELSE 'pending' END,
    updated_at     = now()
  RETURNING chunk_id INTO v_chunk_id;

  -- outbox：内容变更时才入队（未变更无需重算向量）
  IF NOT EXISTS (
       SELECT 1 FROM kb.chunk_embedding
       WHERE chunk_id = v_chunk_id AND domain = p_domain AND model_key = 'bge-m3@v1.5')
     OR (SELECT embed_state FROM kb.chunk WHERE chunk_id = v_chunk_id AND domain = p_domain) = 'pending'
  THEN
    INSERT INTO kb.embedding_job (chunk_id, domain, model_key)
    VALUES (v_chunk_id, p_domain, 'bge-m3@v1.5')
    ON CONFLICT (chunk_id, domain, model_key) DO UPDATE
      SET state = 'queued', attempts = 0, updated_at = now();
  END IF;

  RETURN v_chunk_id;
END;
$$ LANGUAGE plpgsql;
```

### 5.3 发布与回滚（事务 C）

```sql
-- 文档下所有 chunk 都 ready 才原子切换 is_current
CREATE OR REPLACE FUNCTION kb.publish_document(p_doc_id text, p_ingest_run_id uuid)
RETURNS boolean AS $$
DECLARE v_ready boolean;
BEGIN
  SELECT bool_and(embed_state = 'ready')
    INTO v_ready
  FROM kb.chunk
  WHERE doc_id = p_doc_id AND ingest_run_id = p_ingest_run_id;

  IF v_ready IS NULL OR NOT v_ready THEN
    RAISE EXCEPTION 'publish aborted: not all chunks ready for doc %', p_doc_id;
  END IF;

  UPDATE kb.source_document SET is_current = false
   WHERE source_path = (SELECT source_path FROM kb.source_document WHERE doc_id = p_doc_id);
  UPDATE kb.chunk SET is_current = false WHERE doc_id = p_doc_id;

  UPDATE kb.source_document SET is_current = true  WHERE doc_id = p_doc_id;
  UPDATE kb.chunk            SET is_current = true  WHERE doc_id = p_doc_id AND ingest_run_id = p_ingest_run_id;
  RETURN true;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION kb.rollback_document(p_doc_id text, p_prev_run_id uuid)
RETURNS void AS $$
BEGIN
  UPDATE kb.chunk SET is_current = false WHERE doc_id = p_doc_id;
  UPDATE kb.chunk SET is_current = true  WHERE doc_id = p_doc_id AND ingest_run_id = p_prev_run_id;
  UPDATE kb.source_document SET is_current = false WHERE doc_id = p_doc_id;
  UPDATE kb.source_document SET is_current = true
   WHERE source_path = (SELECT source_path FROM kb.source_document WHERE doc_id = p_doc_id)
     AND ingest_run_id = p_prev_run_id;
END;
$$ LANGUAGE plpgsql;
```

### 5.4 is_searchable 同步触发器（保证 partial index 正确）

```sql
CREATE OR REPLACE FUNCTION kb.sync_searchable_flag()
RETURNS trigger AS $$
BEGIN
  IF TG_OP = 'DELETE' THEN
    RETURN OLD;
  END IF;
  -- chunk 任何影响可检索性的字段变化时，同步到同域所有模型的向量行
  UPDATE kb.chunk_embedding em
     SET is_searchable = (NEW.is_current AND NOT NEW.deleted
                          AND NEW.status = 'published' AND NEW.embed_state = 'ready'
                          AND NEW.verified_date IS NOT NULL
                          AND (NEW.expire_date IS NULL OR NEW.expire_date >= CURRENT_DATE))
   WHERE em.chunk_id = NEW.chunk_id AND em.domain = NEW.domain;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_chunk_searchable
AFTER UPDATE OF is_current, deleted, status, embed_state, verified_date, expire_date
ON kb.chunk
FOR EACH ROW EXECUTE FUNCTION kb.sync_searchable_flag();

-- 手动修复函数（D-03 巡检触发）
CREATE OR REPLACE FUNCTION kb.resync_searchable_flag()
RETURNS bigint AS $$
DECLARE v_n bigint;
BEGIN
  UPDATE kb.chunk_embedding em
     SET is_searchable = (c.is_current AND NOT c.deleted
                          AND c.status = 'published' AND c.embed_state = 'ready'
                          AND c.verified_date IS NOT NULL
                          AND (c.expire_date IS NULL OR c.expire_date >= CURRENT_DATE))
  FROM   kb.chunk c
  WHERE  c.chunk_id = em.chunk_id AND c.domain = em.domain;
  GET DIAGNOSTICS v_n = ROW_COUNT;
  RETURN v_n;
END;
$$ LANGUAGE plpgsql;
```

### 5.5 实体/关系 upsert（图抽取 worker 调用）

```sql
CREATE OR REPLACE FUNCTION kb.upsert_entity(
  p_type text, p_canonical text, p_name_norm text, p_description text,
  p_attrs jsonb DEFAULT '{}'::jsonb, p_confidence numeric DEFAULT 1.0)
RETURNS uuid AS $$
DECLARE v_id uuid;
BEGIN
  INSERT INTO kb.entity (entity_type, canonical_name, name_norm, description, attrs, confidence)
  VALUES (p_type, p_canonical, p_name_norm, p_description, p_attrs, p_confidence)
  ON CONFLICT (entity_type, name_norm) DO UPDATE SET
    description = COALESCE(EXCLUDED.description, kb.entity.description),
    confidence  = EXCLUDED.confidence,
    updated_at  = now()
  RETURNING entity_id INTO v_id;
  RETURN v_id;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION kb.upsert_relation(
  p_head uuid, p_type text, p_tail uuid, p_weight numeric DEFAULT 1.0,
  p_confidence numeric DEFAULT 1.0, p_valid_from date DEFAULT NULL)
RETURNS uuid AS $$
DECLARE v_id uuid;
BEGIN
  INSERT INTO kb.relation (head_id, relation_type, tail_id, weight, confidence, valid_from)
  VALUES (p_head, p_type, p_tail, p_weight, p_confidence, COALESCE(p_valid_from, CURRENT_DATE))
  ON CONFLICT (head_id, relation_type, tail_id, valid_from) DO UPDATE SET
    weight = EXCLUDED.weight, confidence = EXCLUDED.confidence, updated_at = now()
  RETURNING relation_id INTO v_id;
  RETURN v_id;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION kb.resync_entity_degree()
RETURNS void AS $$
BEGIN
  UPDATE kb.entity e
     SET degree = COALESCE(x.d, 0)
  FROM (SELECT src AS entity_id, count(*) AS d
          FROM kb.v_graph_edge GROUP BY 1) x
  WHERE x.entity_id = e.entity_id;
END;
$$ LANGUAGE plpgsql;
```

---

## 6. 检索 SQL

### 6.1 Local Search（混合 + 图增强，参数说明见 docs/06 §5.2）

```sql
SET LOCAL hnsw.ef_search       = 100;
SET LOCAL hnsw.iterative_scan  = 'relaxed_order';
SET LOCAL hnsw.max_scan_tuples = 20000;

WITH RECURSIVE hop AS (
    SELECT e.entity_id, 0 AS depth, 1.0::numeric AS path_score,
           ARRAY[e.entity_id] AS path, ARRAY[]::text[] AS rel_path
    FROM   kb.entity e
    WHERE  e.entity_id = ANY ($6) AND NOT e.deleted AND e.status = 'published'
  UNION ALL
    SELECT g.dst, h.depth + 1,
           h.path_score * g.weight * g.confidence * 0.6,
           h.path || g.dst, h.rel_path || g.relation_type
    FROM   hop h
    JOIN   kb.v_graph_edge g ON g.src = h.entity_id
    JOIN   kb.entity te      ON te.entity_id = g.dst
    WHERE  h.depth < 2
      AND  g.relation_type = ANY ($7)
      AND  NOT g.dst = ANY (h.path)
      AND  te.degree <= 500
      AND  h.path_score * g.weight > 0.10
),
dense AS (
    SELECT v.chunk_id, v.domain,
           row_number() OVER (ORDER BY em.embedding <=> $1) AS rnk,
           1 - (em.embedding <=> $1)                        AS cos_sim
    FROM   kb.chunk_embedding em
    JOIN   kb.v_searchable_chunk v ON v.chunk_id = em.chunk_id AND v.domain = em.domain
    WHERE  em.is_searchable AND em.model_key = 'bge-m3@v1.5' AND em.domain = ANY ($3)
      AND  ($4 IS NULL OR v.canteen = $4)
      AND  ($5 IS NULL OR v.business_date IS NULL OR v.business_date = $5)
      AND  ($8 IS NULL OR NOT (v.allergens && $8))
    ORDER  BY em.embedding <=> $1
    LIMIT  30
),
sparse AS (
    SELECT v.chunk_id, v.domain,
           row_number() OVER (ORDER BY ts_rank_cd(v.tsv, q.query) DESC) AS rnk,
           ts_rank_cd(v.tsv, q.query)                                    AS ts_score
    FROM   kb.v_searchable_chunk v,
           websearch_to_tsquery('kb.zh'::regconfig, $2) AS q(query)
    WHERE  v.tsv @@ q.query AND v.domain = ANY ($3)
      AND  ($4 IS NULL OR v.canteen = $4)
      AND  ($5 IS NULL OR v.business_date IS NULL OR v.business_date = $5)
      AND  ($8 IS NULL OR NOT (v.allergens && $8))
    ORDER  BY ts_score DESC
    LIMIT  30
),
graph AS (
    SELECT v.chunk_id, v.domain,
           row_number() OVER (ORDER BY max(h.path_score * ce.salience) DESC) AS rnk,
           max(h.path_score * ce.salience) AS graph_score,
           (array_agg(DISTINCT h.rel_path))[1] AS rel_path
    FROM   hop h
    JOIN   kb.chunk_entity ce ON ce.entity_id = h.entity_id
    JOIN   kb.v_searchable_chunk v ON v.chunk_id = ce.chunk_id AND v.domain = ce.domain
    WHERE  ce.confidence >= 0.70
      AND  ($8 IS NULL OR NOT (v.allergens && $8))
    GROUP  BY v.chunk_id, v.domain
    ORDER  BY graph_score DESC
    LIMIT  20
),
fused AS (
    SELECT chunk_id, domain, sum(w) AS fused_score,
           max(cos_sim) AS cos_sim, max(ts_score) AS ts_score,
           max(graph_score) AS graph_score, max(rel_path) AS rel_path
    FROM (
        SELECT chunk_id, domain, 1.0/(60+rnk)*1.0 AS w, cos_sim, NULL::real ts_score,
               NULL::numeric graph_score, NULL::text[] rel_path FROM dense
        UNION ALL
        SELECT chunk_id, domain, 1.0/(60+rnk)*0.8, NULL, ts_score, NULL, NULL FROM sparse
        UNION ALL
        SELECT chunk_id, domain, 1.0/(60+rnk)*0.6, NULL, NULL, graph_score, rel_path FROM graph
    ) u
    GROUP BY chunk_id, domain
)
SELECT f.chunk_id, f.domain, v.chunk_ref, v.doc_id, v.title, v.breadcrumb, v.content,
       v.authority, v.verified_date, v.source_path, v.canteen, v.floor, v.price,
       f.fused_score, f.cos_sim, f.ts_score, f.graph_score, f.rel_path,
       CASE WHEN f.graph_score IS NOT NULL THEN 'graph_enhanced' ELSE 'vector_or_text' END AS recall_via
FROM   fused f
JOIN   kb.v_searchable_chunk v ON v.chunk_id = f.chunk_id AND v.domain = f.domain
WHERE  NOT EXISTS (
         SELECT 1 FROM fused f2
         JOIN kb.v_searchable_chunk v2 ON v2.chunk_id = f2.chunk_id AND v2.domain = f2.domain
         WHERE v2.canteen  IS NOT DISTINCT FROM v.canteen
           AND v2.category IS NOT DISTINCT FROM v.category
           AND v2.authority = 'primary' AND v.authority <> 'primary'
       )
ORDER  BY f.fused_score DESC
LIMIT  8;
```

### 6.2 Global Search（社区摘要）

```sql
-- ① 社区摘要召回（$1 qvec, $2 level）
SELECT c.community_id, c.level, c.title, c.summary,
       1 - (c.summary_embedding <=> $1) AS sim
FROM   kb.community c
WHERE  c.is_current AND NOT c.stale AND c.level = $2
ORDER  BY c.summary_embedding <=> $1
LIMIT  3;

-- ② 命中社区的高显著度证据 chunk（$3 community_ids）
SELECT DISTINCT v.chunk_ref, v.content, v.verified_date, v.source_path, e.canonical_name
FROM   kb.community_member m
JOIN   kb.entity e        ON e.entity_id = m.entity_id
JOIN   kb.chunk_entity ce ON ce.entity_id = m.entity_id AND ce.salience >= 0.60
JOIN   kb.v_searchable_chunk v ON v.chunk_id = ce.chunk_id AND v.domain = ce.domain
WHERE  m.community_id = ANY ($3)
ORDER  BY v.chunk_ref
LIMIT  20;
```

---

## 7. 巡检 SQL（D/R/G 三类，对应 docs/06 §7）

```sql
-- D-03 冗余标志一致性
SELECT count(*) AS mismatch
FROM   kb.chunk_embedding em
JOIN   kb.chunk c ON c.chunk_id = em.chunk_id AND c.domain = em.domain
WHERE  em.is_searchable <> (c.is_current AND NOT c.deleted
        AND c.status='published' AND c.embed_state='ready'
        AND c.verified_date IS NOT NULL AND (c.expire_date IS NULL OR c.expire_date >= CURRENT_DATE));

-- D-05 ready 但无向量（静默漏召回）
SELECT c.domain, count(*) AS missing
FROM   kb.chunk c
LEFT   JOIN kb.chunk_embedding em ON em.chunk_id=c.chunk_id AND em.domain=c.domain
       AND em.model_key='bge-m3@v1.5'
WHERE  c.embed_state='ready' AND em.chunk_id IS NULL GROUP BY c.domain;

-- D-04 孤儿向量
SELECT count(*) FROM kb.chunk_embedding em
LEFT JOIN kb.chunk c ON c.chunk_id=em.chunk_id AND c.domain=em.domain
WHERE c.chunk_id IS NULL;

-- D-06 模型/维度混用
SELECT model_key, dim, count(*) FROM kb.chunk_embedding GROUP BY 1,2 ORDER BY 3 DESC;

-- D-07 异常范数
SELECT chunk_id, domain, vector_norm(embedding) AS norm
FROM kb.chunk_embedding
WHERE is_normalized AND (vector_norm(embedding) < 0.99 OR vector_norm(embedding) > 1.01);

-- D-08 近似重复（SimHash 汉明距离 <= 3）
SELECT a.chunk_ref, b.chunk_ref, bit_count(a.simhash::bit(64) # b.simhash::bit(64)) AS hamming
FROM kb.chunk a JOIN kb.chunk b ON a.domain=b.domain AND a.chunk_id<b.chunk_id
WHERE a.simhash IS NOT NULL AND b.simhash IS NOT NULL
  AND bit_count(a.simhash::bit(64) # b.simhash::bit(64)) <= 3 LIMIT 100;

-- R-01 HNSW 索引未被使用
SELECT relname, indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE schemaname='kb' AND indexrelname LIKE '%hnsw%' ORDER BY idx_scan;

-- R-03 每个向量分区必须有 HNSW 索引
SELECT c.relname FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace
WHERE n.nspname='kb' AND c.relname LIKE 'chunk_embedding_p_%'
  AND NOT EXISTS (SELECT 1 FROM pg_index i JOIN pg_class ic ON ic.oid=i.indexrelid
                  JOIN pg_am am ON am.oid=ic.relam
                  WHERE i.indrelid=c.oid AND am.amname='hnsw');

-- R-06 索引膨胀
SELECT indexrelname, pg_relation_size(indexrelid) AS idx_size,
       pg_relation_size(indrelid) AS tbl_size
FROM pg_stat_user_indexes WHERE schemaname='kb';

-- G-01 无证据的边
SELECT r.relation_id, r.relation_type FROM kb.relation r
LEFT JOIN kb.relation_evidence e ON e.relation_id=r.relation_id
WHERE e.relation_id IS NULL AND NOT r.deleted;

-- G-03 别名歧义
SELECT alias_norm, count(DISTINCT entity_id), array_agg(DISTINCT entity_id)
FROM kb.entity_alias GROUP BY 1 HAVING count(DISTINCT entity_id) > 1;

-- G-07 degree 漂移
SELECT e.entity_id, e.canonical_name, e.degree AS materialized, x.actual
FROM kb.entity e
JOIN (SELECT src AS entity_id, count(*) AS actual FROM kb.v_graph_edge GROUP BY 1) x USING (entity_id)
WHERE e.degree <> x.actual;

-- 检索偏差看板（近 7 天）
SELECT date_trunc('day', created_at) AS d, count(*) AS queries,
       round(avg(is_empty::int)::numeric,4) AS empty_rate,
       round(avg(top1_score)::numeric,4) AS avg_top1,
       round(avg(graph_hit_cited::int)::numeric,4) AS graph_precision
FROM kb.retrieval_log WHERE created_at >= now()-interval '7 days'
GROUP BY 1 ORDER BY 1 DESC;

-- 域分布异常
SELECT unnest(domains) AS domain, count(*),
       round(100.0*count(*)/sum(count(*)) OVER (),1) AS pct
FROM kb.retrieval_log WHERE created_at >= now()-interval '7 days'
GROUP BY 1 ORDER BY 2 DESC;
```

---

## 8. 权限

```sql
-- 角色
CREATE ROLE kb_ingest NOLOGIN;
CREATE ROLE kb_reader NOLOGIN;
CREATE ROLE kb_ops    NOLOGIN;

-- ingest：基表读写 + 函数执行
GRANT USAGE ON SCHEMA kb TO kb_ingest;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA kb TO kb_ingest;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA kb TO kb_ingest;

-- reader：只读视图
GRANT USAGE ON SCHEMA kb TO kb_reader;
GRANT SELECT ON kb.v_searchable_chunk, kb.v_graph_edge TO kb_reader;

-- ops：巡检与发布回滚函数
GRANT EXECUTE ON FUNCTION kb.publish_document(text, uuid),
                   kb.rollback_document(text, uuid),
                   kb.resync_searchable_flag(), kb.resync_entity_degree() TO kb_ops;

-- 应用账号按需映射（示例）
-- CREATE USER canteen_agent LOGIN PASSWORD '...';
-- GRANT kb_reader TO canteen_agent;
```

---

## 9. zhparser 不可用时的降级

```sql
-- 方案 A：用 simple 配置 + pg_trgm 相似度召回（无需额外扩展）
CREATE TEXT SEARCH CONFIGURATION kb.simple_zh (COPY = simple);

-- 检索 SQL 中 sparse 路改写为 trgm 相似度：
--   WHERE v.content % $2 ORDER BY similarity(v.content, $2) DESC
--   并把融合权重 w_sparse 从 0.8 降为 0.5

-- 方案 B：安装 pg_bigm（2-gram，日文/中文通用）
--   CREATE EXTENSION pg_bigm;
--   CREATE INDEX idx_chunk_bigm ON kb.chunk USING gin (content gin_bigm_ops);
```

---

## 10. 一次性执行清单（按序）

```bash
# 1. 扩展与 schema
psql "$DATABASE_URL" -f 01_extensions.sql
# 2. 枚举/字典/运维表
psql "$DATABASE_URL" -f 02_dict_and_ops.sql
# 3. 主表与分区
psql "$DATABASE_URL" -f 03_core_tables.sql
# 4. 索引（HNSW 构建耗时，建议后台执行）
psql "$DATABASE_URL" -f 04_indexes.sql
# 5. 函数/视图/触发器
psql "$DATABASE_URL" -f 05_functions.sql
# 6. 权限
psql "$DATABASE_URL" -f 06_grants.sql
# 7. 校验：扩展版本与索引存在
psql "$DATABASE_URL" -c "SELECT extversion FROM pg_extension WHERE extname='vector';"
psql "$DATABASE_URL" -c "SELECT indexname FROM pg_indexes WHERE schemaname='kb' AND indexname LIKE '%hnsw%';"
```

> 以上脚本按 §1–§8 顺序拆分保存，命名与本节一致，纳入 `deploy/postgres/` 与迁移工具（Alembic/Flyway）管理。
