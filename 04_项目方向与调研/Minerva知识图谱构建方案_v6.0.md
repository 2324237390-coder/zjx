# Minerva 知识图谱构建方案 v6.0

## 文档版本

| 版本 | 日期 | 说明 |
|------|------|------|
| v5.0 | 2025-06-22 | 架构重构：LLM-First 提取；NER 仅对 source_sentence 执行；Summarizer 移除；UKB 两阶段语义匹配 |
| v6.0 | 2025-06-25 | KG 节点/关系扩展：OpenAlex 四层分类树（Domain→Field→Subfield→Topic）；Author/Disease/Methodology 节点；CITES/AUTHORED_BY/HAS_TOPIC/STUDIES/ADAPTED 关系；Paper 节点修复；Finding 增加 finding_summary |

---

## 零、v5.0 → v6.0 变更摘要

```
v5.0 KG 图:
  Paper ──REPORTS──→ Finding ──has_subject──→ Exposure
                        │
                        └──has_object───→ Outcome

v6.0 KG 图（新增部分用 ★ 标注）:

  Author ──AUTHORED_BY──→ Paper(A) ──CITES──→ Paper(B)
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
  ★ OA_Topic           ★ Disease          ★ Methodology
  (OpenAlex分类)       (ICD-10疾病)       (研究设计/模型)

         │                                       
         ▼                                       
      Finding ──has_subject──→ Exposure          
         │                                       
         └──has_object───→ Outcome               
          【Finding 新增 ★ finding_summary 字段】
          【Exposure/Outcome 归一化方案搁置待议】
```

---

## 一、OpenAlex 四层分类树

### 1.1 数据源

`docs/openalex_filtered_hierarchy.yaml`：OpenAlex 概念的层级结构。

```
Domain (领域)            e.g. "Life Sciences"
  └── Field (学科)        e.g. "Agricultural and Biological Sciences"
        └── Subfield (子学科)  e.g. "General Agricultural and Biological Sciences"
              └── Topic (主题)  e.g. "Agricultural Innovations and Practices"
```

### 1.2 图建模范式

| 节点类型 | 属性 | 索引键 |
|---------|------|--------|
| `OA_Domain` | domain_id (int), domain_name (str) | oa_domain_id |
| `OA_Field` | field_id (int), field_name (str) | oa_field_id |
| `OA_Subfield` | subfield_id (int), subfield_name (str) | oa_subfield_id |
| `OA_Topic` | topic_id (str, e.g. "T10367"), topic_name (str) | oa_topic_id |

### 1.3 关系定义

| 边 | 方向 | 含义 |
|-----|------|------|
| `HAS_FIELD` | OA_Domain → OA_Field | 领域包含学科 |
| `HAS_SUBFIELD` | OA_Field → OA_Subfield | 学科包含子学科 |
| `HAS_TOPIC` | OA_Subfield → OA_Topic | 子学科包含主题 |

注意：`HAS_TOPIC` 与 KG 中 `Paper → OA_Topic` 的关系名不同（后者用 `HAS_TOPIC`，但 Paper → Topic 关系用 `HAS_TOPIC` 会冲突——OA 内部层级和 Paper→Topic 用了同一个名字）。

实际命名方案：
- OA 内部层级关系：`HAS_FIELD` / `HAS_SUBFIELD` / `HAS_TOPIC`
- Paper → OA_Topic：`HAS_TOPIC` — 不冲突，因为起点节点类型不同（OA_Subfield vs Paper），Neo4j 关系类型不区分起点类型。但语义上易混淆，建议 Data Graph 内部层级用 `HAS_TOPIC` 不变，Paper→Topic 用更明确的名称。

**决策：Paper → OA_Topic 关系名为 `HAS_TOPIC`，与 Data Graph 的 `BELONGS_TO` 不冲突（后者已存在但用于不同节点对）。**

```
最终关系命名：
  OA_Subfield ──HAS_TOPIC──→ OA_Topic    （层级关系）
  Paper ──HAS_TOPIC──→ OA_Topic          （论文归类）
  UKB_Field ──BELONGS_TO──→ UKB_Category  （Data Graph, 不动）
```

### 1.4 构建时机

Stage 0 中，先构建 Data Graph，再构建 OpenAlex 分类树。仅需执行一次（节点 MERGE 幂等）。

---

## 二、新增 KG 节点与关系

### 2.1 Author（作者节点）

| 属性 | 类型 | 含义 |
|------|------|------|
| author_id | str | 内部唯一 ID（或 ORCID） |
| name | str | 作者姓名 |
| affiliation | str | 所属机构 |

关系：`Paper -[:AUTHORED_BY]-> Author`

数据源：OpenAlex 记录的 authors 字段 + LLM 元数据提取。

### 2.2 Disease（疾病节点）

| 属性 | 类型 | 含义 |
|------|------|------|
| disease_id | str | ICD-10 编码（如 "E11"） |
| disease_name | str | 标准病名 |
| common_aliases | list[str] | 别名列表 |

关系：`Paper -[:STUDIES]-> Disease`

数据源：scispaCy NER 从论文标题/摘要/全文提取 T047（疾病/综合征）实体，通过 UMLS Linker 获取 ICD-10 编码。

**注意：不从 Outcome 节点派生。** Outcome 当前质量不够稳定，且语义上是"结局变量"而非"疾病研究对象"。Disease 节点独立于 Finding→Outcome 链路。

### 2.3 Methodology（研究方法节点）

| 属性 | 类型 | 含义 |
|------|------|------|
| methodology_id | str | 内部唯一 ID |
| covariates | list[str] | 协变量/校正变量列表 |
| model_type | str | 统计模型（如 "Cox regression"） |
| study_design | str | 研究设计（如 "Cohort Study"） |

关系：`Paper -[:ADAPTED]-> Methodology`

数据源：LLM 提取的 Finding 中已有 evidence_type、adjustment 字段。按 Paper 聚合去重，生成 Methodology 节点。

### 2.4 新增关系汇总

| 关系 | 方向 | 含义 | 数据源 |
|------|------|------|--------|
| `AUTHORED_BY` | Paper → Author | 作者关联 | OpenAlex + LLM |
| `HAS_TOPIC` | Paper → OA_Topic | 论文主题归类 | OpenAlex |
| `STUDIES` | Paper → Disease | 论文研究疾病 | scispaCy NER |
| `ADAPTED` | Paper → Methodology | 论文方法 | LLM Finding 聚合 |
| `CITES` | Paper → Paper | 引用关系 | OpenAlex 引用列表 |

---

## 三、已有节点的修复

### 3.1 Paper 节点

- **删除 `name` 属性**：与 `title` 重复，`_MERGE_PAPER` Cypher 移除 `SET p.name`。
- **修复 `title` 显示 DOI**：当 OpenAlex 无记录时，`title` 回退为 DOI。改为尝试从 PDF 元数据/正文首行/LLM 提取标题。如果确实无法获取标题，title 字段留空而非填入 DOI。

### 3.2 Finding 节点

新增字段：

| 属性 | 类型 | 含义 |
|------|------|------|
| finding_summary | str | 结论的自然语言描述（LLM 生成） |

LLM 提取 prompt 增加 `finding_summary` 输出字段。

### 3.3 Exposure / Outcome

**v6.0 搁置待议。** 当前实现不变。核心问题（UMLS 归一化替换原文、TUI 硬 gate、复合暴露简化）已在讨论中明确方向但未完成设计，后续版本解决。

---

## 四、流水线变更

### Stage 0 扩展

```
Stage 0a: Data Graph（UKB Category + Field）   ← 不变
Stage 0b: OpenAlex Hierarchy Graph              ← ★ 新增
  - 读取 docs/openalex_filtered_hierarchy.yaml
  - MERGE OA_Domain / OA_Field / OA_Subfield / OA_Topic
  - 创建层级关系
```

### Stage 2 扩展

LLM 提取 prompt 增加 `finding_summary` 和 `authors` 字段。

### Stage 8 新增

在 Evaluator → Conflict → Backfill 之后，新增三种关系的写入：

```
Stage 8a: 写入 Author + AUTHORED_BY
Stage 8b: 写入 Disease + STUDIES（NER 提取）
Stage 8c: 写入 Methodology + ADAPTED（聚合 Finding）
Stage 8d: 写入 CITES（OpenAlex）
Stage 9:  Neo4j Ingestion（原有逻辑 + 新关系）
```

---

## 五、项目代码结构

```
minerva/
├── openalex_graph.py # ★ NEW: OpenAlex 分类树构建
├── extractor.py      # ★ 更新: prompt 增加 finding_summary + authors
├── pipeline.py       # ★ 更新: Stage 0b + 新关系写入
├── writer.py         # ★ 更新: 新节点/关系 Cypher, Paper 修复
├── models.py         # ★ 更新: finding_summary, 新 dataclass
├── ner.py            # 不变
├── re.py             # 不变（搁置）
├── evaluator.py      # 不变
├── conflict.py       # 不变
├── backfill.py       # 不变
├── preprocessor.py   # ★ 更新: Paper title 修复
├── ukb_matcher.py    # 不变
├── data_graph.py     # 不变
├── discovery.py      # 不变
├── registry.py       # 不变
├── llm_client.py     # 不变
└── result_logger.py  # 不变
```

---

## 六、技术栈

同 v5.0，新增：

| 环节 | 工具 | 用途 |
|------|------|------|
| OpenAlex 层级 | PyYAML | 解析 filtered_hierarchy.yaml |
| 疾病识别 | scispaCy NER + UMLS Linker（T047 过滤） | 提取论文研究的疾病并 ICD-10 编码 |

---

## 七、修订历史

| 日期 | 版本 | 修订内容 |
|------|------|---------|
| 2025-06-22 | v5.0 | 架构重构：LLM-First；Summarizer 移除；UKB 两阶段语义匹配 |
| 2025-06-25 | v6.0 | OpenAlex 四层分类树；Author/Disease/Methodology 节点；CITES/AUTHORED_BY/HAS_TOPIC/STUDIES/ADAPTED 关系；Paper 修复（去 name、修 title）；Finding 加 finding_summary；Exposure/Outcome 归一化搁置 |
