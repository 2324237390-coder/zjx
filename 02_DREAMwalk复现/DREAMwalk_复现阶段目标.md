# DREAMwalk 复现阶段目标

## 一、复现目标

本次复现 DREAMwalk 的目标不是直接改造 Minerva 知识图谱，也不是马上接入 K-Paths，而是先完成一个最小闭环：

```text
官方示例数据
  → DREAMwalk 官方流程
  → 输出候选 drug-disease 关联
  → 观察候选生成效果
  → 判断是否适合作为 K-Paths 前置候选生成器
```

当前阶段重点是回答三个问题：

1. DREAMwalk 官方代码能否在本地跑通。
2. DREAMwalk 需要什么输入数据，输出什么结果。
3. DREAMwalk 输出的候选 drug-disease pair 是否有接入 K-Paths 的价值。

---

## 二、阶段一：跑通官方 Demo

### 阶段目标

确认 DREAMwalk 官方项目可以在本地环境中正常运行，并能使用官方示例数据输出候选药物-疾病关联结果。

### 小目标

1. 下载 DREAMwalk 官方代码。
2. 确认项目中是否包含：
   - `run_demo.ipynb`
   - `demo/`
   - `environment.yaml`
   - README 或运行说明
3. 阅读 README，记录官方推荐的运行方式。
4. 新建独立 conda 环境，避免影响 K-Paths 和 Minerva 环境。
5. 按 `environment.yaml` 安装依赖。
6. 检查主要 Python 包是否可以正常导入。
7. 运行 `run_demo.ipynb`。
8. 确认 demo 是否能完成：
   - 读取官方示例数据
   - 生成节点表示
   - 输出 drug-disease association prediction
9. 记录运行过程中遇到的报错和解决方式。

### 阶段产出

```text
DREAMwalk 官方 Demo 运行记录
环境配置说明
运行成功或失败截图/日志
demo 输入文件位置
demo 输出文件位置
```

---

## 三、阶段二：理解输入输出格式

### 阶段目标

弄清楚 DREAMwalk 需要什么样的数据，以及它最终输出的候选关联结果长什么样。

### 小目标

1. 查看官方 demo 数据目录。
2. 记录输入文件名称和用途。
3. 明确 DREAMwalk 是否需要以下节点：
   - Drug
   - Gene
   - Disease
4. 明确 DREAMwalk 是否需要以下关系：
   - Drug-Gene
   - Gene-Disease
   - Gene-Gene
   - Drug-Disease
5. 记录每类输入文件的字段含义。
6. 记录实体 ID 的格式。
7. 记录关系边的格式。
8. 记录训练集、测试集或候选预测文件的格式。
9. 查看最终输出结果，确认是否包含：
   - drug
   - disease
   - score
   - ranking
10. 整理一份“DREAMwalk 输入输出格式说明”。

### 阶段产出

```text
DREAMwalk 输入文件清单
DREAMwalk 输出文件清单
节点类型说明
关系类型说明
候选 drug-disease 输出字段说明
```

---

## 四、阶段三：观察候选生成效果

### 阶段目标

判断 DREAMwalk 作为“候选关联生成器”是否有初步价值。

### 小目标

1. 选择 1 至 3 个 demo 中可观察的目标疾病。
2. 查看每个疾病对应的 Top-K 候选药物。
3. 记录 Top-K 候选结果。
4. 区分候选药物中哪些是已知强关联，哪些可能是潜在新候选。
5. 观察候选结果中是否存在明显不合理项。
6. 记录 DREAMwalk 分数与候选排序。
7. 初步判断高分候选是否更合理。
8. 选取 2 至 3 个候选 pair 做 case study。
9. 总结 DREAMwalk 候选生成的优点和问题。

### 阶段产出

```text
DREAMwalk Top-K 候选结果表
候选 drug-disease case study
已知关系与潜在新候选区分记录
候选生成效果初步评价
```

---

## 五、阶段四：判断是否能接入 K-Paths

### 阶段目标

判断 DREAMwalk 输出的候选 drug-disease pair 是否能作为 K-Paths 的输入。

### 小目标

1. 确认 DREAMwalk 输出中是否有 drug ID。
2. 确认 DREAMwalk 输出中是否有 disease ID。
3. 判断这些 ID 是否能和 K-Paths 的节点 ID 对齐。
4. 判断是否能把 DREAMwalk 输出转换为：

```json
{
  "drug_id": "...",
  "disease_id": "...",
  "drug_name": "...",
  "disease_name": "...",
  "score": 0.0
}
```

5. 明确 K-Paths 需要哪些额外信息：
   - 图谱三元组
   - 实体 ID 映射
   - 关系 ID 映射
   - 关系自然语言模板
6. 判断官方 demo 数据是否足够继续跑 K-Paths。
7. 判断如果接入 Minerva，目前缺哪些数据。
8. 输出 DREAMwalk 到 K-Paths 的最小对接方案。

### 阶段产出

```text
DREAMwalk → K-Paths 对接可行性说明
candidate pair 转换格式
K-Paths 所需补充数据清单
Minerva 当前数据缺口清单
```

---

## 六、阶段五：形成复现结论

### 阶段目标

根据前四个阶段的结果，判断 DREAMwalk 是否值得继续作为候选关联生成器使用。

### 小目标

1. 汇总官方 demo 是否跑通。
2. 汇总输入输出格式是否清楚。
3. 汇总候选生成结果是否有意义。
4. 汇总是否能接入 K-Paths。
5. 明确继续推进需要补充的数据。
6. 给出是否继续使用 DREAMwalk 的判断。

### 阶段产出

```text
DREAMwalk 复现总结
是否适合作为候选关联生成器的初步判断
下一步工作建议
```

---

## 七、总体交付物

本次 DREAMwalk 复现完成后，建议形成以下材料：

| 交付物 | 内容 |
|------|------|
| DREAMwalk 运行记录 | 环境、命令、报错、解决方式 |
| 输入输出格式说明 | 官方 demo 数据结构和结果字段 |
| 候选结果观察表 | Top-K drug-disease candidates |
| 对接可行性说明 | DREAMwalk 输出能否进入 K-Paths |
| 数据缺口清单 | Minerva 若要接入 DREAMwalk 需要补什么 |
| 复现总结 | 是否继续采用 DREAMwalk |

