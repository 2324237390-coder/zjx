# 医学知识图谱转 K-Paths 数据

## 1. 转换流程

```text
原始医学知识图谱
-> 实体和关系编码
-> 导出整数三元组
-> 准备待预测实体对
-> 移除目标答案边
-> 抽取 K-Paths
-> 生成 path_str
```

## 2. 必要文件

| 文件 | 内容 |
|---|---|
| `node2id.json` | 业务实体ID到整数节点ID的映射 |
| `id_to_name_mapping.json` | 节点ID对应的实体名称和类型 |
| `relations.json` | 关系ID及自然语言模板 |
| `BKG_file.txt` | 医学知识图谱整数三元组 |
| `train/test.json` | 待预测实体对及评估标签 |

### 实体映射

```json
{
  "DB00001": 101,
  "HGNC:11998": 5001,
  "DOID:10652": 9001
}
```

药物、疾病、基因等实体必须有唯一整数ID，业务数据中的实体必须能映射到图节点。

### 关系映射

```json
{
  "6": "{u} (Drug) binds {v} (Gene)",
  "44": "{u} (Gene) is associated with {v} (Disease)"
}
```

每种关系必须有整数ID、方向和自然语言模板。使用反向边时还要准备反向关系语义。

### 图三元组

将 RDF、Neo4j、CSV 等图谱导出为无表头、空格分隔的：

```text
source_node_id target_node_id relation_id
```

例如：

```text
101 5001 6
5001 9001 44
```

注意：K-Paths 原代码的物理列顺序是 `source target relation`，不是
`source relation target`。

## 3. 待预测实体对

知识图谱之外，还需要提供要预测关系的两个实体：

```json
{
  "entity1_id": 101,
  "entity2_id": 9001,
  "entity1_name": "Drug A",
  "entity2_name": "Disease C",
  "label": "Disease-modifying",
  "label_idx": 0
}
```

`label` 和 `label_idx` 只用于训练或评估，不能作为路径证据输入模型。

## 4. 防止答案泄漏

抽取路径前，必须删除待预测实体之间的目标关系边。

要预测：

```text
Drug A treats Disease C
```

图中不能保留该直接边，但可以保留间接证据：

```text
Drug A binds Gene B
Gene B is associated with Disease C
```

## 5. K-Paths 输出

每个实体对默认抽取最多10条、最长3跳的路径，输出：

| 字段 | 内容 |
|---|---|
| `all_paths` | 节点ID和关系ID路径 |
| `all_paths_str` | 带实体名称和关系名称的结构化路径 |
| `path_str` | 提供给大模型的自然语言证据 |

示例：

```json
{
  "all_paths": [[[101, 6, 5001], [5001, 44, 9001]]],
  "path_str": "Drug A binds Gene B and Gene B is associated with Disease C"
}
```

大模型实际使用的核心输入是 `path_str`。

## 6. 自定义数据集注意事项

官方代码只内置 DrugBank、DDInter 和 PharmacotherapyDB。接入自定义数据集时，
需要在以下代码中增加文件路径、字段名称和标签映射：

```text
k-paths/create_augmented_network.py
k-paths/get-Kpaths.py
k-paths/utils/Kpaths_utils.py
```

## 7. 完成检查

- 查询实体均能映射到整数节点ID。
- 所有关系ID都有自然语言模板。
- 图文件列顺序是 `source target relation`。
- 测试目标边已从图中删除。
- `path_str` 不直接包含真实答案。
- 随机抽查路径具有合理医学含义。
- 统计无路径样本比例，判断知识图谱覆盖是否足够。
